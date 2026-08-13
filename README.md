# OVERVIEW

This repository contains Big Query code using Google Analytics raw data that will provide a summary of am end-to-end e-commerce diagnostic suite that monitors conversion leakage, basket quantity shifts, and stockout bottlenecks across the customer journey.
It transforms raw funnel telemetry into actionable insights, helping teams recover lost revenue and optimise inventory availability

# Repository SQL Scripts Breakdown REPOSITORY SQL SCRIPTS BREAKDOWN 

The repository consists of core SQL scripts in BigQuery organized to power a **4-page Looker Studio Analytics Dashboard**:

1. **`event-funnel-daily-breakdown`** $\rightarrow$ Powers **Page 1: Conversion Funnel & Operational Performance**
2. **`event-funnel-potential-lost-revenue`** $\rightarrow$ Powers **Page 2: Abandonment & Leakage Analysis**
3. **`event-funnel-demand-ceiling`** $\rightarrow$ Powers **Page 3: Inventory & Merchandising Friction**
4. **`event-funnel-cart-to-purchase-changes`** $\rightarrow$ Powers **Page 4: Basket Behavior & Cart Quantity Dynamics**

---

# Key Metrics Overview & Executive Context

Before diving into page-level technical specifications, this section provides an immediate reference for the core financial and conversion metrics tracked across the analytics suite:

### 1. Gross Lost Revenue (In-Stock) (£)
* **What it is:** The sum of all abandoned cart item values for products that are **currently in stock**.
* **Business Context:** Represents high-intent, recoverable revenue. Because the inventory is physically available to fulfill, this metric highlights friction in pricing, UX, or exit intent that can be directly captured through CRO optimizations and automated cart-recovery workflows.

### 2. Unmet Demand Ceiling (Out-of-Stock) (£)
* **What it is:** The total potential monetary demand exposure captured on products that are **out of stock** ($\text{PDP Views} \times \text{Item Price}$).
* **Business Context:** Measures unrecoverable lost revenue caused strictly by supply constraints rather than website performance. Merchandising and purchasing teams use this figure to quantify lost revenue and prioritize vendor restocks based on real user interest.

### 3. Combined Revenue Lost (£)
* **What it is:** The macro sum of all potential cart-stage and checkout-stage drop-off values ($\text{Cart-Stage Lost Revenue} + \text{Checkout Lost Revenue}$).
* **Business Context:** Establishes the top-line "revenue recovery opportunity." It tells executives the total dollar value sitting in non-converting baskets across the entire purchasing journey.

### 4. Cart-Stage Lost Revenue (£)
* **What it is:** Potential revenue lost from users who added items to their cart but **never clicked "Begin Checkout."**
* **Business Context:** Pinpoints top-of-funnel hesitation. High values indicate issues like sudden shipping fee estimates on product pages, lack of trust signals, or an ineffective abandoned cart email flow.

### 5. Checkout Lost Revenue (£)
* **What it is:** Potential revenue lost from high-intent users who **started the checkout process** but failed to complete the purchase.
* **Business Context:** Pinpoints bottom-of-funnel conversion leakage. High values alert technical teams to payment gateway errors, overly complex checkout forms, or missing local payment methods.

### 6. Net Unit & Value Shift (`unit_delta` / `value_delta`)
* **What it is:** The net change between physical units initially added to carts versus final units successfully purchased.
* **Business Context:** Identifies basket modification dynamics. Positive shifts reveal successful cross-selling and bundling, while negative shifts highlight pricing threshold friction where shoppers actively trim items prior to payment.

### 7. Actual Revenue (£)
* **What it is:** The sum of actual item revenue captured from completed purchase events (`event_name = 'purchase'`).
* **Business Context:** Represents actual sales made (cash in the bank). Used across the dashboard as the baseline to evaluate "Actual Revenue vs. Potential/Lost Revenue," showing how much cash was captured compared to what was left in abandoned carts.

## Multi-Day Journey Validation & Test Case

To verify that the dataset accurately handles multi-session consideration cycles, stage separation, and basket trimming without double-counting, a **3-day controlled End-to-End (E2E) test** was executed using two test products (**Product A @ £50** and **Product B @ £70**).

### 📋 3-Day Journey Simulation Setup

| Day | User Action | Direct Event Fired | Net Cart State |
| :--- | :--- | :--- | :--- |
| **Day 1** | Added 2× Product A (£100) + 1× Product B (£70). Then removed 1× Product A and abandoned cart. | `add_to_cart`, `remove_from_cart` | 1× Product A + 1× Product B (£120 total) |
| **Day 2** | Returned to site, added 1× Product B (now 2 units), initiated checkout, filled billing details, then abandoned. | `add_to_cart`, `begin_checkout` | 1× Product A + 2× Product B (£190 total) |
| **Day 3** | Returned to checkout, trimmed Product B back to 1 unit, and completed the order. | `remove_from_cart`, `purchase` | 1× Product A + 1× Product B Purchased (£120 total) |

---

### Metric Reconciliation & Analytical Insights

#### **Page 2: Leakage & Stage Separation Metrics**

| Metric | Recorded Value | Key Insight & Behavioral Interpretation |
| :--- | :--- | :--- |
| **Total Revenue** | **£120** | Captures true completed order value from Day 3 (excluding tax/shipping). |
| **Cart-Stage Lost Revenue** | **£50** | Retains Day 1's initial abandoned cart state prior to checkout initiation. |
| **Checkout Lost Revenue** | **£190** | Accurately isolates mid-funnel leakage when the user entered checkout on Day 2 before abandoning. |
| **Combined Revenue Lost** | **£240** | Sums true non-converted intent across distinct historical sessions without duplicating items. |
| **Cart Abandonment Rate** | **50%** | Reflects multi-day consideration: 1 abandoned cart interaction vs. 1 converted purchase session. |

> **Why This Differs From Legacy Analytics:** Legacy setups would have counted raw cart additions, reporting **>£360+ in lost revenue** by double-counting items moved between cart and checkout stages. The updated model enforces strict stage separation.

---

#### **Page 4: Basket Dynamics & Quantity Shift Metrics**

| Product | Cart Units | Purchased Units | Net Unit Shift | Net Value Shift | Behavior Label |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Product A** (£50) | 1 | 1 | 0 | £0 | `Unchanged` |
| **Product B** (£70) | 2 | 1 | -1 | -£70 | `Quantity Trimmed` |
| **OVERALL TOTAL** | **3** | **2** | **-1** | **-£70** | **Basket Trimming Detected** |

> **Why This Differs From Legacy Analytics:** Traditional GA4 reports only show initial cart views vs final purchases, completely missing mid-funnel quantity edits. This model explicitly highlights **Product B as a "Quantity Trimmed" item**, surfacing price threshold sensitivity where customers scale back quantity immediately before paying.

# Page 1: Conversion Funnel & Operational Performance

### Objective
Tracks daily micro-conversions and step-by-step user movement down the primary ecommerce purchasing funnel:
`Item View` $\rightarrow$ `Add to Cart` $\rightarrow$ `View Cart` $\rightarrow$ `Begin Checkout` $\rightarrow$ `Purchase`.

### Data Source SQL: `event-funnel-daily-breakdown`

### Comprehensive Calculation Matrix (Page 1)

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

# Page 2: Abandonment & Leakage Analysis

### Objective
Isolates financial drop-offs occurring specifically across the funnel, evaluating user conversion windows to identify true cart abandonment vs. checkout leakage.

> **Data Architecture & Attribution Note:**
> * **Net Cart State Accounting:** To prevent inflated abandonment metrics, the underlying SQL computes each item's net cart volume by offsetting `remove_from_cart` events against `add_to_cart` events before evaluating lost revenue.
> * **Stage Separation:** `Cart-Stage Lost Revenue` explicitly excludes items that progressed to `begin_checkout`, ensuring zero double-counting between mid-funnel cart drop-offs and late-stage checkout friction.
> * **Time & Lookback Windowing:** User cart edits and purchases are evaluated within the active reporting window (recommended default: **30 Days**). This allows multi-day customer consideration journeys (e.g., carting on Monday, purchasing on Friday) to properly reconcile within the reporting period.

### Data Source SQL: `event-funnel-potential-lost-revenue`

### Comprehensive Calculation Matrix (Page 2)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Combined Revenue Lost (£)** | `SUM(gross_lost_revenue) + SUM(checkout_lost_revenue)` | Currency (`£`) | Sum of all potential monetary leakage across both cart and checkout stages. | **Macro Opportunity Size:** Top-line financial pipeline lost prior to purchase completion. |
| **Cart-Stage Lost Revenue (£)** | `SUM(cart_units * item_price) WHERE event_name = 'view_cart' AND user_id NOT IN (purchasers)` | Currency (`£`) | Sum of full item potential value present in carts that never reached checkout. | Represents early-stage drop-off (product hesitation, early shipping/fee concerns, or lack of cart-saver emails). |
| **Checkout Lost Revenue (£)** | `SUM(cart_units * item_price) WHERE event_name = 'begin_checkout' AND user_id NOT IN (purchasers)` | Currency (`£`) | Potential value of items in carts where the user initiated checkout but failed to buy. | **Urgent Leakage:** High-intent leads dropping off at final payment/shipping steps. |
| **Category Leakage Share (%)** | `SUM(category_lost_revenue) / NULLIF(SUM(total_lost_revenue), 0)` | Percentage | Category-specific lost revenue divided by total site-wide lost cart value. | Identifies specific merchandise categories suffering from friction or sticker shock. |
| **Cart Abandonment Rate (%)** | `(SUM(cart_users) - SUM(purchasing_users)) / NULLIF(SUM(cart_users), 0)` | Percentage | Unique users who viewed/added to cart minus users who completed purchase, divided by cart users. | Standard ecommerce abandonment metric measuring overall basket drop-off. |
| **Checkout Abandonment Rate (%)** | `(SUM(checkout_users) - SUM(purchasing_users)) / NULLIF(SUM(checkout_users), 0)` | Percentage | Unique users who reached checkout minus actual buyers, divided by checkout starters. | Highlights critical friction occurring exclusively inside the checkout funnel. |

---

# Page 3: Inventory & Merchandising Friction

### Objective
Evaluates supply-chain and stock friction by diagnosing **in-stock cart abandonment** against **out-of-stock demand ceiling potential**, enabling merchandise buyers to prioritize restocks based on actual user traffic.

### Data Source SQL: `event-funnel-demand-ceiling`

### Comprehensive Calculation Matrix (Page 3)

| Metric / Calculated Field | SQL / Looker Studio Formula | Type / Format | Technical Explanation & Logic | Business Meaning & Diagnostic Value |
| :--- | :--- | :--- | :--- | :--- |
| **Gross Lost Revenue (In-Stock)** | `SUM(IF(stock_status != 'outofstock', Abandoned Units * price, 0))` | Currency (`£`) | Total abandoned cart value exclusively for products currently in stock. | High-intent cart leakage that can be directly recovered via CRO/email flows. |
| **Unmet Demand Ceiling (Out-of-Stock)** | `SUM(IF(stock_status = 'outofstock', pdp_views * price, 0))` | Currency (`£`) | Total item value exposure across all PDP views on out-of-stock items (`Views × Price`). | Top-of-funnel merchandising ceiling indicating lost revenue potential due to stockouts. |
| **Actual Revenue** | `SUM(IF(event_name = 'purchase', item_revenue, 0))` | Currency (`£`) | Sum of completed order item revenue captured during the selected period. | Actual sales baseline used to compare realized dollars against lost cart potential. |
| **Abandoned Units** | `GREATEST(0, add_to_cart_count - purchase_count)` | Integer | Subtracts converted units from total carted units per item/date grain. | Physical unit count added to cart but left unpurchased. |
| **Cart Abandonment Rate %** | `SUM(Abandoned Units) / NULLIF(SUM(add_to_cart_count), 0)` | Percentage | Ratio of unpurchased carted items against total cart additions. | Item-level friction metric identifying products with high cart drop-off rates. |

---

# Page 4: Basket Behavior & Cart Quantity Dynamics

### Objective
Analyzes item-level quantity mutability between initial cart creation (`view_cart`) and final order completion (`purchase`), pinpointing items where shoppers expand quantities vs. items trimmed due to price thresholds.

> **Data Architecture & Attribution Note:**
> * **Dynamic Basket Adjustments:** Evaluates true quantity mutability by comparing net cart additions (`add_to_cart` minus `remove_from_cart`) against completed `purchase` quantities at the item-and-date grain.
> * **Behavior Categorization Logic:** Automatically tags user basket interactions into explicit segments:
>   * **Quantity Expanded:** Converted unit count exceeds initial carted units (upselling/bundling success).
>   * **Quantity Trimmed:** Converted unit count is greater than zero but less than initial carted units (price threshold friction).
>   * **Item Removed/Abandoned:** Converted unit count equals zero.
>   * **Unchanged:** Perfect 1:1 unit retention from cart to purchase.
> * **Attribution Horizon:** Captures quantity shifts across multi-session shopping journeys, eliminating false "negative expansion" spikes caused by unadjusted cart views.

### Data Source SQL: `event-funnel-cart-to-purchase-changes`

### Comprehensive Calculation Matrix (Page 4)

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
