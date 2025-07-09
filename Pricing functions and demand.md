### Model Explanation: Demand, Assumptions, and Price Dynamics

This section details the business logic behind the demand-based pricing model, the key assumptions it relies on, and how the final price responds to changes in market conditions.

---

#### 1. The Demand Function

The model does not use a single, simple variable for demand. Instead, it constructs a composite **`total_demand` score** by synthesizing four key factors. This score is a weighted average that represents the overall pressure on the service at a given time.

The core formula is:

```
total_demand = (0.50 * core_demand) +
               (0.20 * queue_pressure) +
               (0.15 * special_day_surge) +
               (0.15 * traffic_multiplier_effect)
```

Each component is engineered to capture a specific real-world phenomenon:

*   **Core Demand (50% weight):** This is the primary driver, based on the `occupancy_ratio`. It uses a **sigmoid (S-curve) function**.
    *   **Logic:** The impact of one additional customer is not linear. When the system is nearly empty, one more user has little effect on demand. Similarly, when the system is almost full, it's already in a high-demand state. The greatest impact occurs around the **50% occupancy mark**, where the system transitions from "available" to "busy." The sigmoid function models this non-linear, rapid transition.

*   **Queue Pressure (20% weight):** This factor models the negative experience and visible demand pressure of a waiting line. It uses a **scaled hyperbolic tangent (tanh) function**.
    *   **Logic:** Like occupancy, the impact of a queue is non-linear. The first few people forming a queue dramatically increase perceived demand. However, the difference between a 20-person queue and a 25-person queue is less significant. The `tanh` function captures this "diminishing returns" effect, where the pressure grows quickly at first and then levels off.

*   **Special Day Surge (15% weight):** This is a binary factor that applies a significant, fixed increase to the demand score on designated special days (e.g., holidays, major events).
    *   **Logic:** This accounts for predictable, external events that are guaranteed to increase demand, regardless of the current occupancy. It acts as a predefined surge multiplier.

*   **Traffic Multiplier (15% weight):** This factor linearly increases the demand score based on the intensity of nearby traffic.
    *   **Logic:** High traffic is a proxy for general congestion and high activity in the area, suggesting that demand for local services (like parking or transportation) will also be higher.

In summary, the demand function is a sophisticated, weighted combination of internal metrics (occupancy, queues) and external signals (special days, traffic) to create a holistic measure of real-time demand.

---

#### 2. Key Assumptions

Every model is a simplification of reality. This pricing model operates on the following key assumptions:

1.  **Peak Values are Representative:** The model aggregates daily data by taking the `max()` of metrics like `Occupancy` and `QueueLength`. It assumes that the **peak demand** during a day is the most important signal for setting a daily price profile. It implicitly ignores the duration of that peak (e.g., a 10-minute spike is treated the same as a 4-hour surge).

2.  **The Factors are Additive and Independent:** The model calculates the `total_demand` score as a weighted sum. This assumes that the influence of occupancy, queues, and traffic can be reasonably added together. It does not explicitly model complex interactions (e.g., how extreme traffic might directly *cause* a queue to form).

3.  **The Weights and Formulas are Correct:** The weights (0.50, 0.20, etc.) and mathematical functions (sigmoid, tanh) are hand-picked based on business expertise. The model assumes these parameters accurately reflect the relative importance of each factor and their real-world behavior. They are not learned from historical data via machine learning.

4.  **Competition is an Indirect Factor:** The model has no direct input for `competitor_price` or `competitor_occupancy`. It assumes that the actions of competitors will be **implicitly reflected** in our own data. For example, if a competitor lowers their price, our occupancy might fall, which the model will then react to. It is a *reactive* model, not a *proactive* one in its competitive strategy.

5.  **Vehicle Premiums are Static:** The surcharges and discounts for different vehicle types (+25% for trucks, -15% for bikes) are fixed. The model assumes this premium/discount structure is always appropriate and does not need to change based on the overall demand level.

---

#### 3. How Price Changes with Demand and Competition

The model's pricing is designed to be responsive, smooth, and bounded.

**Response to Demand:**

When the `total_demand` score increases, the price rises in a controlled, two-step process:

1.  **Raw Price Calculation:** First, a `raw_price` is calculated which increases linearly with the `total_demand` score.
2.  **Final Price Smoothing:** This `raw_price` is then passed through a final sigmoid function to calculate the actual `price`. This has crucial effects:
    *   **Price Floor & Ceiling:** The price is bounded. It will never fall below a minimum (**$5.00**) or exceed a maximum (**$20.00**). This provides predictability for customers and the business.
    *   **Smooth Transitions:** The sigmoid curve ensures that price changes are smooth, not sudden. The price rises most steeply around the midpoint of its range (when `raw_price` is near 12.5) and levels off as it approaches the floor or ceiling, preventing extreme price shocks.

**Response to Competition:**

As noted in the assumptions, the model responds to competition **indirectly by observing its effects on our own metrics**.

*   **Scenario 1: A competitor lowers their price.**
    1.  Customers are drawn to the competitor.
    2.  Our `Occupancy` and `QueueLength` decrease.
    3.  Our `core_demand` and `queue_pressure` scores fall.
    4.  Our `total_demand` score decreases.
    5.  **Result:** Our model automatically lowers our price to remain competitive.

*   **Scenario 2: A competitor's facility fills up or closes.**
    1.  Customers are diverted to our service.
    2.  Our `Occupancy` and `QueueLength` increase.
    3.  Our `core_demand` and `queue_pressure` scores rise.
    4.  Our `total_demand` score increases.
    5.  **Result:** Our model automatically raises our price in response to the increased market demand.

In this way, the model is inherently **market-reactive**. While it doesn't explicitly "see" competitors, it dynamically adjusts to the flow of customers, which is the ultimate outcome of all competitive actions.
