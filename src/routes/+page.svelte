<script lang="ts">
	import { createClient } from '@supabase/supabase-js';
	import { onMount } from 'svelte';

	type StatementType = 'income' | 'expense';
	type FilterMode = 'day' | 'month' | 'year' | 'all';
	type RawRow = Record<string, unknown>;

	type Statement = {
		id: string;
		created_at: string;
		title: string;
		detail: string;
		type: StatementType;
		category: string;
		amount: number;
		cost: number;
	};

	type Sale = {
		id: string;
		created_at: string;
		receipt_no: string;
		subtotal: number;
		tax_amount: number;
		discount: number;
		total_amount: number;
		payment_method: string;
		note: string;
		taxpayer_name: string;
		taxpayer_id: string;
		sale_date: string;
		status: string;
	};

	type Expense = {
		id: string;
		created_at: string;
		category: string;
		details: string;
		amount: number;
		tax_amount: number;
		payment_method: string;
		line_user_id: string;
		receipt_image: string;
		expense_date: string;
	};

	type Product = {
		id: string;
		created_at: string;
		product_name: string;
		category: string;
		cost_price: number;
		sell_price: number;
		stock_qty: number;
		minimum_stock: number;
		unit: string;
		active: boolean;
	};

	type SaleItem = {
		id: string;
		created_at: string;
		quantity: number;
		unit_price: number;
		total_price: number;
		cost_price: number;
		profit: number;
		sale_id: string;
		product_id: string;
	};

	type StockMovement = {
		id: string;
		created_at: string;
		movement_type: string;
		quantity: number;
		productName: string;
		note: string;
	};

	const supabaseUrl = 'https://xalmbmjnsrpjjsbnowfo.supabase.co';
	const supabaseAnonKey =
		'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InhhbG1ibWpuc3JwampzYm5vd2ZvIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODEwMTU3NzksImV4cCI6MjA5NjU5MTc3OX0.7qIEAfjQ9htpmDzl-_xOV1XpukfEcSOic88pBhKJ1RY';
	const supabase = createClient(supabaseUrl, supabaseAnonKey);

	let sales = $state<Sale[]>([]);
	let expenses = $state<Expense[]>([]);
	let products = $state<Product[]>([]);
	let saleItems = $state<SaleItem[]>([]);
	let stockMovements = $state<StockMovement[]>([]);
	let statements = $state<Statement[]>([]);
	let selectedProductId = $state('');
	let unitCostInput = $state(0);
	let sellPrice = $state(0);
	let stockQtyInput = $state(0);
	let loading = $state(true);
	let errorMessage = $state('');
	let productSaveMessage = $state('');
	let savingProduct = $state(false);
	let filterMode = $state<FilterMode>('month');
	let taxRate = $state(5);

	const filteredStatements = $derived(statements.filter((item) => inSelectedPeriod(item, filterMode)));
	const summary = $derived(summarize(filteredStatements, taxRate));
	const periodSummary = $derived(buildPeriodSummary(statements));
	const costItems = $derived(
		expenses
			.filter((item) =>
				inSelectedPeriod(
					{
						id: `expense-${item.id}`,
						created_at: item.created_at,
						title: item.category || 'Expense',
						detail: item.details,
						type: 'expense',
						category: item.category || 'Expense',
						amount: item.amount + item.tax_amount,
						cost: 0
					},
					filterMode
				)
			)
			.sort((a, b) => b.amount - a.amount)
	);
	const inventoryValue = $derived(
		products.reduce((sum, product) => sum + product.stock_qty * Math.max(product.cost_price, 0), 0)
	);
	const totalStockQty = $derived(products.reduce((sum, product) => sum + product.stock_qty, 0));
	const lowStockProducts = $derived(
		products
			.filter((product) => product.minimum_stock > 0 && product.stock_qty <= product.minimum_stock)
			.sort((a, b) => a.stock_qty - b.stock_qty)
	);
	const allProducts = $derived(
		[...products].sort((a, b) => a.product_name.localeCompare(b.product_name, 'th'))
	);
	const selectedProduct = $derived(
		products.find((product) => product.id === selectedProductId) ?? products[0] ?? null
	);
	const profitCalculator = $derived(
		buildProfitCalculator(selectedProduct, unitCostInput, sellPrice, stockQtyInput)
	);
	const barBase = $derived(
		Math.max(summary.totalIncome, summary.totalExpense + summary.totalCost, 1)
	);
	const recentMovements = $derived(
		[...stockMovements].sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
	);
	const recentSaleItems = $derived(
		[...saleItems].sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime())
	);

	onMount(async () => {
		try {
			const [salesResult, expensesResult, productsResult, saleItemsResult, movementsResult] = await Promise.all([
				fetchTable('sales', 'created_at, id, receipt_no, subtotal, tax_amount, discount, total_amount, payment_method, note, taxpayer_name, taxpayer_id, sale_date, status'),
				fetchTable('expenses', 'id, created_at, category, details, amount, tax_amount, payment_method, line_user_id, receipt_image, expense_date'),
				fetchTable('products', 'created_at, id, product_name, category, cost_price, sell_price, stock_qty, minimum_stock, unit, active'),
				fetchTable('sale_item', 'id, created_at, quantity, unit_price, total_price, cost_price, profit, sale_id, product_id'),
				fetchTable('stock_movements', 'created_at, movement_type, quantity, id, product_id, reference_id, note')
			]);

			sales = salesResult.data.map(normalizeSale);
			expenses = expensesResult.data.map(normalizeExpense);
			products = productsResult.data.map(normalizeProduct);
			saleItems = saleItemsResult.data.map(normalizeSaleItem);
			stockMovements = movementsResult.data.map((item) => normalizeStockMovement(item, products));
			if (!selectedProductId && products.length > 0) {
				applyProductPreset(products[0].id);
			}

			const errorParts = [salesResult.error, expensesResult.error, productsResult.error, saleItemsResult.error, movementsResult.error]
				.map((item) => (item ? item.message : ''))
				.filter(Boolean);
			errorMessage = errorParts.join(' | ');

			statements = buildStatements(sales, expenses, saleItems);
		} catch (error) {
			errorMessage =
				error instanceof Error
					? `เชื่อมต่อ Supabase ไม่สำเร็จ: ${error.message}`
					: 'เชื่อมต่อ Supabase ไม่สำเร็จ';
		} finally {
			loading = false;
		}
	});

	async function fetchTable<T extends RawRow>(table: string, columns = '*') {
		const query = supabase.from(table).select(columns, { count: 'exact' });
		const { data, error, count } = table === 'products'
			? await query.order('created_at', { ascending: false })
			: await query.order('created_at', { ascending: false });
		return {
			data: (data ?? []) as unknown as T[],
			count: count ?? 0,
			error
		};
	}

	function buildStatements(salesRows: Sale[], expenseRows: Expense[], saleItemRows: SaleItem[]): Statement[] {
		const costBySaleId = new Map<string, number>();

		for (const item of saleItemRows) {
			if (!item.sale_id) continue;
			const quantity = Math.max(item.quantity, 1);
			const lineCost = item.cost_price > 0 ? item.cost_price * quantity : Math.max(item.total_price - item.profit, 0);
			const currentCost = costBySaleId.get(item.sale_id) ?? 0;
			costBySaleId.set(item.sale_id, currentCost + lineCost);
		}

		const incomeStatements: Statement[] = salesRows.map((sale) => ({
			id: `sale-${sale.id}`,
			created_at: sale.created_at,
			title: sale.receipt_no ? `บิล ${sale.receipt_no}` : `ยอดขาย ${shortId(sale.id)}`,
			detail: [sale.status, sale.payment_method, sale.note].filter(Boolean).join(' · ') || 'บันทึกการขาย',
			type: 'income',
			category: 'Sales',
			amount: sale.total_amount > 0 ? sale.total_amount : sale.subtotal,
			cost: costBySaleId.get(sale.id) ?? 0
		}));

		const expenseStatements: Statement[] = expenseRows.map((expense) => ({
			id: `expense-${expense.id}`,
			created_at: expense.created_at,
			title: expense.category || 'Expense',
			detail: [expense.details, expense.payment_method].filter(Boolean).join(' · ') || 'บันทึกรายจ่าย',
			type: 'expense',
			category: expense.category || 'Expense',
			amount: expense.amount + expense.tax_amount,
			cost: 0
		}));

		return [...incomeStatements, ...expenseStatements].sort(
			(a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
		);
	}

	function normalizeSale(item: RawRow): Sale {
		return {
			id: String(item.id ?? ''),
			created_at: String(item.sale_date ?? item.created_at ?? new Date().toISOString()),
			receipt_no: String(item.receipt_no ?? ''),
			subtotal: toNumber(item.subtotal),
			tax_amount: toNumber(item.tax_amount),
			discount: toNumber(item.discount),
			total_amount: toNumber(item.total_amount),
			payment_method: String(item.payment_method ?? ''),
			note: String(item.note ?? ''),
			taxpayer_name: String(item.taxpayer_name ?? ''),
			taxpayer_id: String(item.taxpayer_id ?? ''),
			sale_date: String(item.sale_date ?? ''),
			status: String(item.status ?? '')
		};
	}

	function normalizeExpense(item: RawRow): Expense {
		return {
			id: String(item.id ?? ''),
			created_at: String(item.expense_date ?? item.created_at ?? new Date().toISOString()),
			category: String(item.category ?? ''),
			details: String(item.details ?? ''),
			amount: toNumber(item.amount),
			tax_amount: toNumber(item.tax_amount),
			payment_method: String(item.payment_method ?? ''),
			line_user_id: String(item.line_user_id ?? ''),
			receipt_image: String(item.receipt_image ?? ''),
			expense_date: String(item.expense_date ?? '')
		};
	}

	function normalizeProduct(item: RawRow): Product {
		return {
			id: String(item.id ?? ''),
			created_at: String(item.created_at ?? new Date().toISOString()),
			product_name: String(item.product_name ?? 'ไม่ระบุชื่อสินค้า'),
			category: String(item.category ?? 'Uncategorized'),
			cost_price: toNumber(item.cost_price),
			sell_price: toNumber(item.sell_price),
			stock_qty: toNumber(item.stock_qty),
			minimum_stock: toNumber(item.minimum_stock),
			unit: String(item.unit ?? ''),
			active: Boolean(item.active)
		};
	}

	function normalizeSaleItem(item: RawRow): SaleItem {
		return {
			id: String(item.id ?? ''),
			created_at: String(item.created_at ?? new Date().toISOString()),
			quantity: toNumber(item.quantity),
			unit_price: toNumber(item.unit_price),
			total_price: toNumber(item.total_price),
			cost_price: toNumber(item.cost_price),
			profit: toNumber(item.profit),
			sale_id: String(item.sale_id ?? ''),
			product_id: String(item.product_id ?? '')
		};
	}

	function normalizeStockMovement(item: RawRow, productRows: Product[]): StockMovement {
		const product = productRows.find((entry) => entry.id === String(item.product_id ?? ''));
		return {
			id: String(item.id ?? ''),
			created_at: String(item.created_at ?? new Date().toISOString()),
			movement_type: String(item.movement_type ?? ''),
			quantity: toNumber(item.quantity),
			productName: product?.product_name ?? `สินค้า ${shortId(String(item.product_id ?? item.id ?? ''))}`,
			note: String(item.note ?? '')
		};
	}

	function applyProductPreset(productId: string) {
		selectedProductId = productId;
		const product = products.find((entry) => entry.id === productId);
		if (!product) return;
		unitCostInput = product.cost_price;
		sellPrice = product.sell_price;
		stockQtyInput = 0;
		productSaveMessage = '';
	}

	async function persistSelectedProduct() {
		if (!selectedProductId) return;

		savingProduct = true;
		productSaveMessage = '';

		const payload = {
			cost_price: Math.max(unitCostInput, 0),
			sell_price: Math.max(sellPrice, 0),
			stock_qty: Math.max((selectedProduct?.stock_qty ?? 0) + Math.max(stockQtyInput, 0), 0)
		};

		const { data, error } = await supabase
			.from('products')
			.update(payload)
			.eq('id', selectedProductId)
			.select('created_at, id, product_name, category, cost_price, sell_price, stock_qty, minimum_stock, unit, active')
			.single();

		savingProduct = false;

		if (error) {
			productSaveMessage = `อัปเดตสินค้าไม่สำเร็จ: ${error.message}`;
			return;
		}

		const updatedProduct = normalizeProduct(data as unknown as RawRow);
		products = products.map((product) => (product.id === updatedProduct.id ? updatedProduct : product));
		applyProductPreset(updatedProduct.id);
		productSaveMessage = 'บันทึกข้อมูลสินค้าไปที่ Supabase แล้ว';
	}

	function buildProfitCalculator(
		product: Product | null,
		unitCostValue: number,
		unitSellPrice: number,
		stockQtyValue: number
	) {
		const unitCost = Math.max(unitCostValue || product?.cost_price || 0, 0);
		const sellingPrice = Math.max(unitSellPrice, 0);
		const currentStockQty = Math.max(product?.stock_qty || 0, 0);
		const stockQty = Math.max(currentStockQty + Math.max(stockQtyValue, 0), 0);
		const grossProfit = sellingPrice - unitCost;
		const margin = sellingPrice > 0 ? (grossProfit / sellingPrice) * 100 : 0;
		const markUp = unitCost > 0 ? ((sellingPrice - unitCost) / unitCost) * 100 : 0;
		const stockValue = stockQty * unitCost;

		return {
			unitCost,
			sellingPrice,
			currentStockQty,
			stockQty,
			grossProfit,
			margin,
			markUp,
			stockValue
		};
	}

	function summarize(items: Statement[], rate: number) {
		const totalIncome = items
			.filter((item) => item.type === 'income')
			.reduce((sum, item) => sum + item.amount, 0);
		const totalCost = items
			.filter((item) => item.type === 'income')
			.reduce((sum, item) => sum + item.cost, 0);
		const totalExpense = items
			.filter((item) => item.type === 'expense')
			.reduce((sum, item) => sum + item.amount, 0);
		const netProfit = totalIncome - totalCost - totalExpense;
		const estimatedTax = netProfit > 0 ? netProfit * (rate / 100) : 0;

		return {
			totalIncome,
			totalCost,
			totalExpense,
			netProfit,
			estimatedTax,
			cashFlow: totalIncome - totalExpense,
			profitMargin: totalIncome > 0 ? (netProfit / totalIncome) * 100 : 0
		};
	}

	function buildPeriodSummary(items: Statement[]) {
		return (['day', 'month', 'year'] as FilterMode[]).map((mode) => ({
			mode,
			label: mode === 'day' ? 'วันนี้' : mode === 'month' ? 'เดือนนี้' : 'ปีนี้',
			...summarize(
				items.filter((item) => inSelectedPeriod(item, mode)),
				taxRate
			)
		}));
	}

	function inSelectedPeriod(item: Statement, mode: FilterMode) {
		if (mode === 'all') return true;

		const date = parseDateValue(item.created_at);
		const now = new Date();

		if (Number.isNaN(date.getTime())) return false;
		if (mode === 'day') {
			return (
				date.getDate() === now.getDate() &&
				date.getMonth() === now.getMonth() &&
				date.getFullYear() === now.getFullYear()
			);
		}
		if (mode === 'month') return date.getMonth() === now.getMonth() && date.getFullYear() === now.getFullYear();
		return date.getFullYear() === now.getFullYear();
	}

	function parseDateValue(value: string) {
		const text = String(value ?? '').trim();
		if (/^\d{4}-\d{2}-\d{2}$/.test(text)) {
			const [year, month, day] = text.split('-').map(Number);
			return new Date(year, month - 1, day);
		}
		return new Date(text);
	}

	function toNumber(value: unknown) {
		const parsed = Number(String(value ?? 0).replace(/,/g, ''));
		return Number.isFinite(parsed) ? parsed : 0;
	}

	function money(value: number) {
		return new Intl.NumberFormat('th-TH', {
			style: 'currency',
			currency: 'THB',
			maximumFractionDigits: 0
		}).format(value);
	}

	function dateText(value: string) {
		return new Intl.DateTimeFormat('th-TH', {
			day: '2-digit',
			month: 'short',
			year: 'numeric'
		}).format(parseDateValue(value));
	}

	function filterLabel(mode: FilterMode) {
		if (mode === 'day') return 'รายวัน';
		if (mode === 'month') return 'รายเดือน';
		if (mode === 'year') return 'รายปี';
		return 'ทั้งหมด';
	}

	function movementLabel(value: string) {
		const text = value.toLowerCase();
		if (text.includes('in')) return 'รับเข้า';
		if (text.includes('out')) return 'จ่ายออก';
		if (text.includes('adjust')) return 'ปรับสต็อก';
		return value || 'ไม่ระบุ';
	}

	function shortId(value: string | number | null | undefined) {
		const text = String(value ?? '').replace(/-/g, '').trim();
		return text.slice(0, 6).toUpperCase() || 'NEW';
	}
</script>

<svelte:head>
	<title>Statement Dashboard</title>
</svelte:head>

<main class="shell">
	<header class="topbar">
		<div>
			<p class="eyebrow">Supabase statement</p>
			<h1>สรุปรายรับ รายจ่าย ต้นทุน และภาษี</h1>
		</div>

		<div class="controls" aria-label="ตัวกรองช่วงเวลา">
			{#each ['day', 'month', 'year', 'all'] as mode}
				<button class:active={filterMode === mode} onclick={() => (filterMode = mode as FilterMode)}>
					{filterLabel(mode as FilterMode)}
				</button>
			{/each}
		</div>
	</header>

	{#if loading}
		<section class="state-panel">กำลังโหลดข้อมูลจาก Supabase...</section>
	{:else}
		{#if errorMessage}
			<section class="notice">
				<strong>ข้อมูลบางตารางโหลดไม่ครบ</strong>
				<span>{errorMessage}</span>
			</section>
		{/if}

		<section class="summary-grid" aria-label="สรุปยอด">
			<article class="metric income">
				<span>รายรับรวม</span>
				<strong>{money(summary.totalIncome)}</strong>
				<small>{filteredStatements.filter((item) => item.type === 'income').length} รายการขายจาก `sales`</small>
			</article>
			<article class="metric cost">
				<span>ต้นทุนขาย</span>
				<strong>{money(summary.totalCost)}</strong>
				<small>คำนวณจาก `sale_item.cost_price`</small>
			</article>
			<article class="metric stock">
				<span>มูลค่าสต็อก</span>
				<strong>{money(inventoryValue)}</strong>
				<small>รวมจาก `stock_qty x cost_price` ของทุกสินค้า</small>
			</article>
			<article class="metric expense">
				<span>รายจ่ายทั่วไป</span>
				<strong>{money(summary.totalExpense)}</strong>
				<small>{filteredStatements.filter((item) => item.type === 'expense').length} รายการจาก `expenses`</small>
			</article>
			<article class="metric profit" class:negative={summary.netProfit <= 0}>
				<span>กำไรสุทธิ</span>
				<strong>{money(summary.netProfit)}</strong>
				<small>Margin {summary.profitMargin.toFixed(1)}%</small>
			</article>
		</section>

		<section class="tax-panel">
			<div>
				<p class="eyebrow">Estimated tax</p>
				<h2>ประมาณการภาษีจากกำไรสุทธิ</h2>
				<p>อัตราภาษี {taxRate}% จากกำไรสุทธิหลังหักต้นทุนและรายจ่าย</p>
			</div>
			<label>
				<span>อัตราภาษี (%)</span>
				<input type="number" min="0" max="35" step="0.5" bind:value={taxRate} />
			</label>
			<strong>{money(summary.estimatedTax)}</strong>
		</section>

		<section class="period-grid" aria-label="สรุปตามช่วงเวลา">
			{#each periodSummary as period}
				<article>
					<span>{period.label}</span>
					<strong>{money(period.netProfit)}</strong>
					<small>รับ {money(period.totalIncome)} · จ่าย {money(period.totalExpense + period.totalCost)}</small>
				</article>
			{/each}
		</section>

		<section class="single-layout">
			<div class="panel">
				<div class="section-heading">
					<div>
						<p class="eyebrow">Cost check</p>
						<h2>เช็ครายการต้นทุน</h2>
					</div>
					<span>{costItems.length} รายการ</span>
				</div>

				{#if costItems.length === 0}
					<p class="empty">ยังไม่มีรายการต้นทุนในช่วง {filterLabel(filterMode)}</p>
				{:else}
					<div class="cost-list">
						{#each costItems.slice(0, 6) as item (item.id)}
							<div class="cost-row">
								<div>
									<strong>{item.category || 'Expense'}</strong>
									<span>{item.details || 'บันทึกรายจ่าย'} · {dateText(item.created_at)}</span>
								</div>
								<b>{money(item.amount + item.tax_amount)}</b>
							</div>
						{/each}
					</div>
				{/if}
				<p class="helper-text">ดึงข้อมูลตรงจากตาราง `expenses` ตามช่วงเวลาที่เลือก</p>
			</div>

			<div class="panel">
				<div class="section-heading">
					<div>
						<p class="eyebrow">Cash flow</p>
						<h2>สรุปกระแสเงินสด</h2>
					</div>
					<span>{filterLabel(filterMode)}</span>
				</div>
				<div class="bars">
					<div class="bar-row">
						<span>รายรับ</span>
						<meter class="bar income-bar" min="0" max={barBase} value={summary.totalIncome}></meter>
						<b>{money(summary.totalIncome)}</b>
					</div>
					<div class="bar-row">
						<span>จ่าย+ต้นทุน</span>
						<meter class="bar expense-bar" min="0" max={barBase} value={summary.totalExpense + summary.totalCost}></meter>
						<b>{money(summary.totalExpense + summary.totalCost)}</b>
					</div>
				</div>
			</div>
		</section>

		<section class="single-layout">
			<div class="panel">
				<div class="section-heading">
					<div>
						<p class="eyebrow">Products</p>
						<h2>ชื่อสินค้า หมวด และสต็อก</h2>
					</div>
					<span>{allProducts.length} รายการ</span>
				</div>
				{#if allProducts.length === 0}
					<p class="empty">ยังไม่มีสินค้าจากตาราง `products`</p>
				{:else}
					<div class="table-wrap">
						<table class="compact-table products-table">
							<thead>
								<tr>
									<th>ชื่อสินค้า</th>
									<th>หมวด</th>
									<th>ต้นทุน</th>
									<th>ราคาขาย</th>
									<th>สต็อก</th>
									<th>จำนวนสินค้าขั้นต่ำ</th>
								</tr>
							</thead>
							<tbody>
								{#each allProducts.slice(0, 8) as product (product.id)}
									<tr>
										<td class="product-name-cell">
											<strong>{product.product_name}</strong>
											<span>{product.unit || 'unit'}</span>
										</td>
										<td>{product.category}</td>
										<td>{money(product.cost_price)}</td>
										<td>{money(product.sell_price)}</td>
										<td>{product.stock_qty}</td>
										<td>{product.minimum_stock}</td>
									</tr>
								{/each}
							</tbody>
						</table>
					</div>
				{/if}
			</div>
		</section>

		<section class="panel calculator-panel">
			<div class="section-heading">
				<div>
					<p class="eyebrow">Profit calculator</p>
					<h2>คำนวณต้นทุน-กำไรตามรอบสินค้า</h2>
				</div>
				{#if selectedProduct}
					<span>{selectedProduct.product_name}</span>
				{/if}
			</div>

			{#if !selectedProduct}
				<p class="empty">ยังไม่มีสินค้าให้คำนวณ</p>
			{:else}
				<div class="calculator-grid">
					<div class="calculator-form">
						<label class="field">
							<span>สินค้า</span>
							<select bind:value={selectedProductId} onchange={(event) => applyProductPreset((event.currentTarget as HTMLSelectElement).value)}>
								{#each allProducts as product (product.id)}
									<option value={product.id}>{product.product_name}</option>
								{/each}
							</select>
						</label>
						<label class="field">
							<span>ราคาต้นทุนต่อหน่วย</span>
							<input
								type="number"
								min="0"
								step="0.01"
								bind:value={unitCostInput}
								onchange={persistSelectedProduct}
							/>
						</label>
						<label class="field">
							<span>ราคาขาย</span>
							<input
								type="number"
								min="0"
								step="0.01"
								bind:value={sellPrice}
								onchange={persistSelectedProduct}
							/>
						</label>
						<label class="field">
							<span>เพิ่มสต็อก</span>
							<input
								type="number"
								min="0"
								step="1"
								bind:value={stockQtyInput}
								onchange={persistSelectedProduct}
							/>
						</label>
					</div>

					<div class="calculator-summary">
						<div class="calc-hero" class:negative={profitCalculator.margin <= 10}>
							<span>กำไรขั้นต้น</span>
							<strong class:plus={profitCalculator.grossProfit > 0} class:minus={profitCalculator.grossProfit <= 0}>
								{profitCalculator.grossProfit > 0 ? '+' : '-'}{money(Math.abs(profitCalculator.grossProfit))}
							</strong>
							<small class:plus={profitCalculator.margin > 10} class:minus={profitCalculator.margin <= 10}>
								{profitCalculator.margin > 0 ? '+' : '-'}{Math.abs(profitCalculator.margin).toFixed(1)}%
							</small>
						</div>

						<div class="calc-grid">
							<article>
								<span>ต้นทุนต่อชิ้น</span>
								<strong>{money(profitCalculator.unitCost)}</strong>
							</article>
							<article>
								<span>ราคาขายต่อชิ้น</span>
								<strong>{money(profitCalculator.sellingPrice)}</strong>
							</article>
							<article>
								<span>กำไรต่อชิ้น</span>
								<strong class:plus={profitCalculator.grossProfit > 0} class:minus={profitCalculator.grossProfit <= 0}>
									{profitCalculator.grossProfit > 0 ? '+' : '-'}{money(Math.abs(profitCalculator.grossProfit))}
								</strong>
							</article>
							<article>
								<span>Markup จากต้นทุน</span>
								<strong class:plus={profitCalculator.markUp > 0} class:minus={profitCalculator.markUp <= 0}>
									{profitCalculator.markUp > 0 ? '+' : '-'}{Math.abs(profitCalculator.markUp).toFixed(1)}%
								</strong>
							</article>
							<article>
								<span>สต็อกปัจจุบัน</span>
								<strong>{profitCalculator.currentStockQty} ชิ้น</strong>
							</article>
							<article>
								<span>สต็อกหลังอัปเดต</span>
								<strong>{profitCalculator.stockQty} ชิ้น</strong>
							</article>
							<article>
								<span>มูลค่าสต็อกตามต้นทุนใหม่</span>
								<strong>{money(profitCalculator.stockValue)}</strong>
							</article>
						</div>

						<p class="helper-text">
							ช่องสต็อกทำงานแบบเพิ่มจากของเดิม ถ้าใส่ 10 และสินค้าเดิมมี 200 ระบบจะบันทึกเป็น 210 ในตาราง `products` แล้วรีเฟรชตารางสินค้าให้ทันที
						</p>
						{#if savingProduct || productSaveMessage}
							<p class="helper-text">{savingProduct ? 'กำลังบันทึกข้อมูลสินค้า...' : productSaveMessage}</p>
						{/if}
					</div>
				</div>
			{/if}
		</section>

		<section class="panel table-panel">
			<div class="section-heading">
				<div>
					<p class="eyebrow">Sales detail</p>
					<h2>รายการขายและรายจ่าย</h2>
				</div>
				<span>{filteredStatements.length} รายการ</span>
			</div>

			{#if filteredStatements.length === 0}
				<p class="empty">ไม่มีข้อมูลในช่วงเวลานี้</p>
			{:else}
				<div class="table-wrap">
					<table>
						<thead>
							<tr>
								<th>วันที่</th>
								<th>รายการ</th>
								<th>ประเภท</th>
								<th>หมวด</th>
								<th>ยอดเงิน</th>
							</tr>
						</thead>
						<tbody>
							{#each filteredStatements as item (item.id)}
								<tr>
									<td>{dateText(item.created_at)}</td>
									<td>
										<strong>{item.title}</strong>
										{#if item.detail}
											<span>{item.detail}</span>
										{/if}
									</td>
									<td>
										<mark class={item.type}>{item.type === 'income' ? 'รายรับ' : 'รายจ่าย'}</mark>
									</td>
									<td>{item.category}</td>
									<td class:plus={item.type === 'income'} class:minus={item.type === 'expense'}>
										{item.type === 'income' ? '+' : '-'}{money(item.amount)}
									</td>
								</tr>
							{/each}
						</tbody>
					</table>
				</div>
			{/if}
		</section>

		<section class="panel saleitem-panel">
			<div class="section-heading">
				<div>
					<p class="eyebrow">sale_item</p>
					<h2>รายการขายย่อยที่ใช้คำนวณต้นทุน</h2>
				</div>
				<span>{recentSaleItems.length} รายการ</span>
			</div>

			{#if recentSaleItems.length === 0}
				<p class="empty">ยังไม่มีรายการจาก `sale_item`</p>
			{:else}
				<div class="table-wrap">
					<table class="compact-table">
						<thead>
							<tr>
								<th>วันที่</th>
								<th>Sale ID</th>
								<th>จำนวน</th>
								<th>ต้นทุนต่อชิ้น</th>
								<th>กำไร</th>
							</tr>
						</thead>
						<tbody>
							{#each recentSaleItems.slice(0, 8) as item (item.id)}
								<tr>
									<td>{dateText(item.created_at)}</td>
									<td>{shortId(item.sale_id)}</td>
									<td>{item.quantity}</td>
									<td>{money(item.cost_price)}</td>
									<td>{money(item.profit)}</td>
								</tr>
							{/each}
						</tbody>
					</table>
				</div>
			{/if}
		</section>
	{/if}
</main>

<style>
	:global(body) {
		margin: 0;
		--orange: #E04A22;
		--amber: #E04A22;
		--sky: #21519B;
		--orange-strong: #E04A22;
		--ink: #111111;
		--muted: rgba(17, 17, 17, 0.58);
		--line: rgba(17, 17, 17, 0.1);
		--surface: #ffffff;
		--orange-soft: #fff4ee;
		--orange-soft-2: #ffe9df;
		--blue-soft: #eef4ff;
		--panel-tint: linear-gradient(180deg, #fff3ec 0%, #fffdfb 100%);
		min-height: 100vh;
		background:
			radial-gradient(circle at top left, #ffe5d8 0%, transparent 28%),
			radial-gradient(circle at top right, #dfeaff 0%, transparent 26%),
			linear-gradient(180deg, #fff7f2 0%, #ffffff 100%);
		color: var(--ink);
		font-family:
			Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
	}

	.shell {
		width: min(1380px, calc(100% - 28px));
		margin: 12px auto 24px;
		padding: 18px 14px 36px;
		border: 0;
		border-radius: 20px;
		background: linear-gradient(180deg, #fffdfc 0%, #ffffff 100%);
		box-shadow: none;
		backdrop-filter: none;
	}

	.topbar,
	.section-heading,
	.tax-panel {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 20px;
	}

	h1,
	h2,
	p {
		margin: 0;
	}

	h1 {
		margin-top: 6px;
		font-size: clamp(1.95rem, 3.1vw, 3.05rem);
		line-height: 1.03;
		letter-spacing: 0;
		font-weight: 800;
		max-width: none;
		white-space: nowrap;
	}

	h2 {
		font-size: 1.08rem;
	}

	.eyebrow {
		color: var(--muted);
		font-size: 0.74rem;
		font-weight: 800;
		text-transform: uppercase;
		letter-spacing: 0.04em;
	}

	.topbar {
		padding: 10px 6px 18px;
		border-bottom: 0;
		background: linear-gradient(90deg, #fff3ec 0%, #ffffff 52%, #eff4ff 100%);
		border-radius: 24px;
		padding-left: 18px;
		padding-right: 18px;
	}

	.topbar > div:first-child {
		display: grid;
		gap: 4px;
	}

	.controls {
		display: flex;
		min-height: 48px;
		padding: 6px;
		border: 1px solid rgba(17, 17, 17, 0.06);
		border-radius: 999px;
		background: linear-gradient(90deg, #fff0e8 0%, #ffffff 100%);
		box-shadow:
			inset 0 1px 0 rgba(255, 255, 255, 0.9),
			0 8px 24px rgba(224, 74, 34, 0.08);
	}

	button {
		min-width: 84px;
		border: 0;
		border-radius: 999px;
		background: transparent;
		color: var(--ink);
		font: inherit;
		font-weight: 700;
		padding: 0 16px;
		cursor: pointer;
		transition:
			background 160ms ease,
			color 160ms ease,
			transform 160ms ease,
			box-shadow 160ms ease;
	}

	button.active {
		background: var(--orange-strong);
		color: white;
		box-shadow: 0 12px 24px rgba(224, 74, 34, 0.22);
		transform: translateY(-1px);
	}

	.summary-grid,
	.period-grid,
	.split-layout,
	.single-layout {
		display: grid;
		gap: 16px;
		margin-top: 20px;
	}

	.summary-grid {
		grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
		margin-top: 26px;
	}

	.period-grid {
		grid-template-columns: repeat(3, minmax(0, 1fr));
	}

	.split-layout {
		grid-template-columns: 1fr 1fr;
	}

	.single-layout {
		grid-template-columns: 1fr;
	}

	.metric,
	.period-grid article,
	.panel,
	.tax-panel,
	.notice,
	.state-panel {
		border: 1px solid var(--line);
		border-radius: 24px;
		background: var(--panel-tint);
		box-shadow:
			0 18px 44px rgba(17, 17, 17, 0.045),
			0 10px 24px rgba(224, 74, 34, 0.05);
	}

	.metric,
	.period-grid article {
		padding: 22px;
	}

	.metric {
		display: grid;
		align-content: start;
		min-height: 164px;
		position: relative;
		overflow: hidden;
	}

	.metric span,
	.period-grid span {
		display: block;
		color: var(--muted);
		font-size: 0.86rem;
		font-weight: 700;
	}

	.metric strong,
	.period-grid strong {
		display: block;
		margin-top: 12px;
		font-size: 2.15rem;
		line-height: 0.96;
	}

	.metric small,
	.period-grid small {
		display: block;
		margin-top: 12px;
		color: var(--muted);
		line-height: 1.4;
	}

	.period-grid article {
		border: 1px solid #efc3b4;
		border-top: 4px solid #e04a22;
		background: linear-gradient(135deg, #fff0e8 0%, #ffffff 100%);
	}

	.period-grid article span,
	.period-grid article strong,
	.period-grid article small {
		color: var(--ink);
	}

	.income {
		border-top: 4px solid var(--sky);
		background: linear-gradient(135deg, #2f5ea8 0%, #4f79bc 100%);
	}

	.cost {
		border-top: 4px solid var(--amber);
		background: linear-gradient(135deg, #f18f52 0%, #e4552b 100%);
	}

	.stock {
		border-top: 4px solid var(--orange-strong);
		background: linear-gradient(135deg, #ef8246 0%, #e25128 100%);
	}

	.expense {
		border-top: 4px solid var(--orange-strong);
		background: linear-gradient(135deg, #f29258 0%, #e35a30 100%);
	}

	.profit {
		border-top: 4px solid var(--sky);
		background: linear-gradient(135deg, #f39762 0%, #e45e36 100%);
	}

	.metric::after {
		content: "";
		position: absolute;
		right: -18px;
		bottom: -18px;
		width: 96px;
		height: 96px;
		border-radius: 28px;
		background: linear-gradient(135deg, rgba(255, 255, 255, 0.12), rgba(255, 255, 255, 0.02));
		transform: rotate(12deg);
	}

	.income span,
	.income strong,
	.income small,
	.cost span,
	.cost strong,
	.cost small,
	.stock span,
	.stock strong,
	.stock small,
	.expense span,
	.expense strong,
	.expense small,
	.profit span,
	.profit strong,
	.profit small {
		color: #ffffff;
	}

	.income small,
	.cost small,
	.stock small,
	.expense small,
	.profit small {
		opacity: 0.98;
	}

	.profit.negative {
		border-top-color: var(--orange);
	}

	.tax-panel,
	.panel,
	.notice,
	.state-panel {
		margin-top: 18px;
		padding: 24px 26px;
	}

	.tax-panel {
		display: grid;
		grid-template-columns: minmax(0, 1fr) auto auto;
		align-items: center;
		gap: 18px;
		border: 1px solid #efc3b4;
		border-top: 4px solid #e04a22;
		background: linear-gradient(135deg, #fff0e8 0%, #ffffff 70%, #eef4ff 100%);
	}

	.tax-panel p {
		margin-top: 6px;
		color: var(--muted);
	}

	.tax-panel label {
		display: grid;
		gap: 6px;
		color: var(--ink);
		font-size: 0.82rem;
		font-weight: 700;
	}

	.tax-panel .eyebrow,
	.tax-panel h2,
	.tax-panel > strong {
		color: var(--ink);
	}

	input {
		width: 96px;
		box-sizing: border-box;
		border: 1px solid var(--line);
		border-radius: 6px;
		padding: 10px 12px;
		background: #ffffff;
		color: var(--ink);
		font: inherit;
		font-weight: 800;
	}

	select {
		width: 100%;
		box-sizing: border-box;
		border: 1px solid var(--line);
		border-radius: 6px;
		padding: 10px 12px;
		background: #ffffff;
		color: var(--ink);
		font: inherit;
		font-weight: 700;
	}

	.tax-panel > strong {
		min-width: 160px;
		text-align: right;
		font-size: 1.7rem;
	}

	.notice {
		display: grid;
		gap: 6px;
		border-color: var(--amber);
		background: linear-gradient(135deg, #ffece3 0%, #fffdfc 100%);
		color: var(--ink);
	}

	.state-panel,
	.empty {
		color: var(--muted);
		text-align: center;
	}

	.section-heading {
		margin-bottom: 20px;
	}

	.panel {
		padding: 24px 26px;
	}

	.section-heading > span {
		border: 1px solid rgba(224, 74, 34, 0.18);
		border-radius: 999px;
		padding: 9px 14px;
		background: linear-gradient(135deg, #ffe5d8 0%, #fff5ef 100%);
		color: var(--orange);
		font-size: 0.82rem;
		font-weight: 800;
		white-space: nowrap;
	}

	.cost-list,
	.inventory-list {
		display: grid;
		gap: 10px;
	}

	.calculator-grid {
		display: grid;
		grid-template-columns: minmax(280px, 360px) minmax(0, 1fr);
		gap: 18px;
	}

	.calculator-form {
		display: grid;
		gap: 14px;
		padding: 22px;
		border: 1px solid rgba(17, 17, 17, 0.08);
		border-radius: 20px;
		background: linear-gradient(180deg, #fff3ec 0%, #ffffff 100%);
	}

	.field {
		display: grid;
		gap: 6px;
	}

	.field span {
		color: var(--muted);
		font-size: 0.82rem;
		font-weight: 800;
	}

	.field input {
		width: 100%;
	}

	.calculator-summary {
		display: grid;
		gap: 16px;
	}

	.calc-hero {
		padding: 24px;
		border: 1px solid #1f8f45;
		border-radius: 22px;
		background: linear-gradient(135deg, rgba(49, 180, 92, 0.34) 0%, rgba(255, 255, 255, 0.98) 100%);
	}

	.calc-hero.negative {
		border-color: #cf2f2f;
		background: linear-gradient(135deg, rgba(239, 83, 80, 0.34) 0%, rgba(255, 255, 255, 0.98) 100%);
	}

	.calc-hero span {
		display: block;
		color: var(--ink);
		font-size: 0.82rem;
		font-weight: 800;
	}

	.calc-hero strong {
		display: block;
		margin-top: 8px;
		font-size: clamp(2rem, 5vw, 3rem);
		line-height: 1;
	}

	.calc-hero small {
		display: block;
		margin-top: 10px;
		font-size: 1rem;
		font-weight: 800;
		color: var(--ink);
	}

	.calc-hero strong {
		color: var(--ink);
	}

	.calc-grid {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 12px;
	}

	.calc-grid article {
		padding: 18px;
		border: 1px solid #efc3b4;
		border-radius: 18px;
		background: linear-gradient(180deg, #fff2ea 0%, #ffffff 100%);
	}

	.calc-grid span {
		display: block;
		color: var(--muted);
		font-size: 0.8rem;
		font-weight: 800;
	}

	.calc-grid strong {
		display: block;
		margin-top: 8px;
		font-size: 1.3rem;
		line-height: 1.2;
		color: var(--ink);
	}

	.cost-row,
	.inventory-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 16px;
		padding: 15px 0;
		border-top: 1px solid rgba(17, 17, 17, 0.08);
	}

	.cost-row div {
		display: grid;
		gap: 4px;
	}

	.cost-row span,
	td span,
	.helper-text {
		color: var(--muted);
		font-size: 0.86rem;
	}

	.helper-text {
		margin-top: 12px;
	}

	.bars {
		display: grid;
		gap: 18px;
	}

	.bar-row {
		display: grid;
		grid-template-columns: 90px 1fr 120px;
		align-items: center;
		gap: 12px;
	}

	.bars span {
		color: var(--muted);
		font-weight: 800;
	}

	.bar {
		width: 100%;
		height: 12px;
		border: 0;
		border-radius: 999px;
		background: linear-gradient(90deg, #ffe8dc 0%, #eef4ff 100%);
		overflow: hidden;
	}

	.bar::-webkit-meter-bar {
		background: rgba(17, 17, 17, 0.06);
		border: 0;
		border-radius: 999px;
	}

	.bar::-webkit-meter-optimum-value {
		background: var(--sky);
		border-radius: 999px;
	}

	.bar::-moz-meter-bar {
		background: var(--sky);
		border-radius: 999px;
	}

	.expense-bar::-webkit-meter-optimum-value {
		background: var(--orange-strong);
	}

	.expense-bar::-moz-meter-bar {
		background: var(--orange-strong);
	}

	.bars b,
	.cost-row b {
		text-align: right;
	}

	.table-panel,
	.saleitem-panel {
		padding: 0;
		overflow: hidden;
	}

	.table-panel .section-heading,
	.saleitem-panel .section-heading {
		margin: 0;
		padding: 24px 26px;
		border-bottom: 1px solid rgba(17, 17, 17, 0.08);
		background: linear-gradient(90deg, #fff1e8 0%, #ffffff 58%, #eef4ff 100%);
	}

	.table-wrap {
		overflow-x: auto;
	}

	table {
		width: 100%;
		min-width: 820px;
		border-collapse: collapse;
		background: #ffffff;
	}

	.compact-table {
		min-width: 0;
	}

	.products-table {
		table-layout: fixed;
	}

	.products-table th:first-child,
	.products-table td:first-child {
		width: 46%;
	}

	.products-table th:nth-child(2),
	.products-table td:nth-child(2) {
		width: 14%;
	}

	.products-table th:nth-child(3),
	.products-table td:nth-child(3),
	.products-table th:nth-child(4),
	.products-table td:nth-child(4) {
		width: 12%;
	}

	.products-table th:nth-child(5),
	.products-table td:nth-child(5),
	.products-table th:nth-child(6),
	.products-table td:nth-child(6) {
		width: 8%;
	}

	.product-name-cell strong {
		display: block;
		line-height: 1.3;
		word-break: break-word;
	}

	.product-name-cell span {
		display: block;
		margin-top: 4px;
	}

	th,
	td {
		padding: 16px 20px;
		border-bottom: 1px solid rgba(17, 17, 17, 0.08);
		text-align: left;
		vertical-align: top;
	}

	th {
		background: linear-gradient(180deg, #fff0e7 0%, #f8fbff 100%);
		color: var(--ink);
		font-size: 0.78rem;
		text-transform: uppercase;
	}

	tbody tr {
		transition: background 160ms ease;
	}

	tbody tr:hover {
		background: linear-gradient(90deg, #fff7f1 0%, #fffdfb 100%);
	}

	.panel:nth-of-type(odd):not(.table-panel):not(.saleitem-panel):not(.calculator-panel) {
		background: linear-gradient(180deg, #fff6f0 0%, #ffffff 100%);
	}

	.panel:nth-of-type(even):not(.table-panel):not(.saleitem-panel):not(.calculator-panel) {
		background: linear-gradient(180deg, #f7faff 0%, #ffffff 100%);
	}

	td strong {
		display: block;
		margin-bottom: 4px;
	}

	td:last-child,
	th:last-child {
		text-align: right;
		white-space: nowrap;
		font-weight: 800;
	}

	mark {
		border-radius: 999px;
		padding: 6px 11px;
		font-size: 0.78rem;
		font-weight: 800;
	}

	mark.income {
		background: rgba(33, 81, 155, 0.14);
		color: #111111;
	}

	mark.expense {
		background: rgba(224, 74, 34, 0.14);
		color: #111111;
	}

	.plus {
		color: rgba(31, 143, 69, 0.96);
	}

	.minus {
		color: rgba(207, 47, 47, 0.96);
	}

	@media (max-width: 900px) {
		.topbar,
		.tax-panel,
		.section-heading {
			align-items: flex-start;
			flex-direction: column;
		}

		.summary-grid,
		.period-grid,
		.split-layout,
		.single-layout {
			grid-template-columns: 1fr;
		}

		.controls {
			width: 100%;
			overflow-x: auto;
		}

		button {
			flex: 1;
		}

		.tax-panel > strong {
			text-align: left;
		}

		.calculator-grid {
			grid-template-columns: 1fr;
		}
	}

	@media (max-width: 560px) {
		.shell {
			width: min(100% - 12px, 1380px);
			margin-top: 8px;
			padding: 14px 6px 24px;
			border-radius: 16px;
		}

		h1 {
			font-size: 2rem;
			white-space: normal;
		}

		.controls {
			display: grid;
			grid-template-columns: repeat(2, 1fr);
		}

		button {
			min-height: 40px;
		}

		.bar-row {
			grid-template-columns: 1fr;
		}

		.bars b {
			text-align: left;
		}

		.calc-grid {
			grid-template-columns: 1fr;
		}
	}
</style>
