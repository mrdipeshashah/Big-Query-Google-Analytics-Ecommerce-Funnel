# 📁 Repository SQL Scripts Breakdown

The repository consists of core SQL scripts in BigQuery organized to power a **4-page Looker Studio Analytics Dashboard**:

1. **`event-funnel-daily-breakdown`** $\rightarrow$ Powers **Page 1: Conversion Funnel & Operational Performance**
2. **`event-funnel-potential-lost-revenue`** $\rightarrow$ Powers **Page 2: Abandonment & Leakage Analysis**
3. **`event-funnel-demand-ceiling`** $\rightarrow$ Powers **Page 3: Inventory & Merchandising Friction**
4. **`event-funnel-cart-to-purchase-changes`** $\rightarrow$ Powers **Page 4: Basket Behavior & Cart Quantity Dynamics**

---

# 📄 Page 1: Conversion Funnel & Operational Performance

### Objective
Tracks daily micro-conversions and step-by-step user movement down the primary ecommerce purchasing funnel:
`Item View` $\rightarrow$ `Add to Cart` $\rightarrow$ `View Cart` $\rightarrow$ `Begin Checkout` $\rightarrow$ `Purchase`.

### Data Source SQL: `event-funnel-daily-breakdown`

### 🧮 Comprehensive Calculation Matrix (Page 1)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Total Event Volume** | `COUNT(event_name)` | Integer | Raw count of all captured funnel events within the selected date window. | High-level traffic and interaction pulse across the site. |
| **Unique Funnel Users** | `COUNT(DISTINCT user_pseudo_id)` | Integer | Count of unique client IDs completing at least one funnel action. | Evaluates actual user reach rather than repeat event spam. |
| **Item View to Cart Rate (%)** | `SUM(add_to_cart_count) / NULLIF(SUM(view_item_count), 0)` | Percentage | Ratios total cart additions against product detail page views. | Measures product page conversion power and CTA effectiveness. |
| **Cart to Checkout Rate (%)** | `SUM(begin_checkout_count) / NULLIF(SUM(view_cart_count), 0)` | Percentage | Ratios users entering checkout against users reviewing their cart. | Evaluates friction between intent and actual checkout commitment. |
| **Checkout Completion Rate (%)** | `SUM(purchase_count) / NULLIF(SUM(begin_checkout_count), 0)` | Percentage | Ratios final completed orders against checkout initiations. | Measures payment gateway, shipping fee, and checkout form efficiency. |
| **Overall Funnel Conversion Rate (%)** | `SUM(purchase_count) / NULLIF(SUM(view_item_count), 0)` | Percentage | End-to-end conversion efficiency from initial interest to final order. | Main macro efficiency KPI for ecommerce operations and growth marketing. |
| **Macro Drop-Off Rate (%)** | `1 - (SUM(purchase_count) / NULLIF(SUM(view_item_count), 0))` | Percentage | Complement of the overall funnel conversion rate. | Quantifies the total share of site visitors lost across all funnel stages. |

---

# 📄 Page 2: Abandonment & Leakage Analysis

### Objective
Isolates financial drop-offs occurring specifically across the funnel, evaluating user conversion windows to identify true cart abandonment vs. checkout leakage.

### Data Source SQL: `event-funnel-potential-lost-revenue`

### 🧮 Comprehensive Calculation Matrix (Page 2)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Gross Lost Revenue (£)** | `SUM(cart_units * item_price) WHERE event_name = 'view_cart' AND user_id NOT IN (purchasers)` | Currency (`£`) | Sum of full item potential value present in carts that never resulted in a purchase event. | Total top-line monetary pipeline that entered the cart but leaked away. |
| **High-Intent Lost Revenue (£)** | `SUM(cart_units * item_price) WHERE event_name = 'begin_checkout' AND user_id NOT IN (purchasers)` | Currency (`£`) | Potential value of items in carts where the user initiated checkout but failed to buy. | **Urgent Leakage:** Represents warm leads losing out at the payment/shipping stage. |
| **Category Leakage Share (%)** | `SUM(category_lost_revenue) / NULLIF(SUM(total_lost_revenue), 0)` | Percentage | Category-specific lost revenue divided by total site-wide lost cart value. | Identifies specific merchandise categories suffering from friction or sticker shock. |
| **Cart Abandonment Rate (%)** | `(SUM(cart_users) - SUM(purchasing_users)) / NULLIF(SUM(cart_users), 0)` | Percentage | Unique users who viewed/added to cart minus users who completed purchase, divided by cart users. | Standard ecommerce abandonment metric measuring overall basket drop-off. |
| **Checkout Abandonment Rate (%)** | `(SUM(checkout_users) - SUM(purchasing_users)) / NULLIF(SUM(checkout_users), 0)` | Percentage | Unique users who reached checkout minus actual buyers, divided by checkout starters. | Highlights critical friction occurring exclusively inside the checkout funnel. |

---

# 📄 Page 3: Inventory & Merchandising Friction

### Objective
Evaluates supply-chain and stock friction by diagnosing **in-stock cart abandonment** against **out-of-stock demand ceiling potential**, enabling merchandise buyers to prioritize restocks based on actual user traffic.

### Data Source SQL: `event-funnel-demand-ceiling`

### 🧮 Comprehensive Calculation Matrix (Page 3)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Gross Lost Revenue (In-Stock)** | `SUM(IF(stock_status != 'outofstock', Abandoned Units * price, 0))` | Currency (`£`) | Total abandoned cart value exclusively for products currently in stock. | High-intent cart leakage that can be directly recovered via CRO/email flows. |
| **Unmet Demand Ceiling (Out-of-Stock)** | `SUM(IF(stock_status = 'outofstock', pdp_views * price, 0))` | Currency (`£`) | Total item value exposure across all PDP views on out-of-stock items (`Views × Price`). | Top-of-funnel merchandising ceiling indicating lost revenue potential due to stockouts. |
| **Realized Revenue** | `SUM(IF(event_name = 'purchase', item_revenue, 0))` | Currency (`£`) | Sum of completed order item revenue captured during the selected period. | Actual sales baseline used to compare realized dollars against lost cart potential. |
| **Abandoned Units** | `GREATEST(0, add_to_cart_count - purchase_count)` | Integer | Subtracts converted units from total carted units per item/date grain. | Physical unit count added to cart but left unpurchased. |
| **Cart Abandonment Rate %** | `SUM(Abandoned Units) / NULLIF(SUM(add_to_cart_count), 0)` | Percentage | Ratio of unpurchased carted items against total cart additions. | Item-level friction metric identifying products with high cart drop-off rates. |

---

# 📄 Page 4: Basket Behavior & Cart Quantity Dynamics

### Objective
Analyzes item-level quantity mutability between initial cart creation (`view_cart`) and final order completion (`purchase`), pinpointing items where shoppers expand quantities vs. items trimmed due to price thresholds.

### Data Source SQL: `event-funnel-cart-to-purchase-changes`

### 🧮 Comprehensive Calculation Matrix (Page 4)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Initial Cart Volume (`cart_units`)** | `SUM(c.cart_units)` | Integer | Total units aggregated from `view_cart` events grouped by `item_id` and date. | Total physical unit volume initially intended for purchase by customers. |
| **Purchased Volume (`purchased_units`)** | `SUM(p.purchased_units)` | Integer | Total units aggregated from `purchase` events grouped by `item_id` and date. | Total physical unit volume successfully converted into actual sales. |
| **Net Unit Shift (`unit_delta`)** | `COALESCE(purchased_units, 0) - COALESCE(cart_units, 0)` | Integer | Direct subtraction: purchased units minus carted units. Positive = Expansion, Negative = Trimming. | Measures whether customers add more items or reduce quantities before buying. |
| **Net Revenue Shift (`value_delta`)** | `COALESCE(purchased_value, 0) - COALESCE(cart_value, 0)` | Currency (`£`) | Monetary difference between final order item value and initial carted item value. | Net financial impact (£) resulting from basket adjustments prior to order completion. |
| **Net Unit Shift %** | `SUM(unit_delta) / NULLIF(SUM(cart_units), 0)` | Percentage | Ratios the total unit difference against the original carted volume. | Percentage expansion or contraction of overall physical basket size. |
| **Net Value Shift %** | `SUM(value_delta) / NULLIF(SUM(cart_value), 0)` | Percentage | Ratios the monetary revenue delta against the initial potential cart value. | Realized value growth/decay percentage from cart building to order placement. |
| **Unit Retention Rate (%)** | `SUM(purchased_units) / NULLIF(SUM(cart_units), 0)` | Percentage | Percentage of original carted units that survived through to final purchase. | Benchmark metric for basket retention (e.g., 100% = no net unit change). |
| **Cart Behavior Classification (`cart_behavior_type`)** | `CASE WHEN p_units > c_units THEN 'Quantity Expanded' WHEN p_units < c_units AND p_units > 0 THEN 'Quantity Trimmed' WHEN p_units = 0 THEN 'Item Removed/Abandoned' ELSE 'Unchanged' END` | Categorical Dimension | Categorizes item interaction based on unit changes between cart and purchase. | Categorizes customer basket modification behavior for segmented analysis. |
