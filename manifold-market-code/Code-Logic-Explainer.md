# Maniswap

### Reference

- [Maniswap v2](bit.ly/maniswap)
- [Maniswap v3](https://manifoldmarkets.notion.site/Maniswap-V3-f619ffcb5cd540888fc31d164446a952)

To calculate the change in probability after a bet and the maximum payout in Manifold Markets' modified CPMM model, let's summarize the key components and work through the steps in detail.

### **Overview of the Manifold CPMM Model**

The model you shared is based on the following relationships:

1. **Market Equation**:
   $$
   y^p \cdot n^{1-p} = k
   $$
   Where:

   - $y$ = number of YES shares in the pool
   - $n$ = number of NO shares in the pool
   - $p$ = a market parameter related to the probability
   - $k$ = a constant, determined by the liquidity and shares in the pool

2. __Market Probability Formula__:
   The current probability of YES ($P_{YES}$) is:
   $$
   P_{YES} = \frac{p \cdot n}{p \cdot n + (1 - p) \cdot y}
   $$

### **Change in Probability After the Bet**

To calculate the __new probability__ after a bet, we need to consider how the shares in the pool change due to the bet and how the new $p$ (denoted as $p_{\text{new}}$) is recalculated to maintain the market probability.

Let's break this down:

1. **Initial Market Setup**:

   - You have the number of YES and NO shares in the pool: $y$ and $n$, respectively.
   - The current probability is given by:
      $$
      P_{YES} = \frac{p \cdot n}{p \cdot n + (1 - p) \cdot y}
      $$

2. **Post-Bet Adjustment**:
   
   Assume a bet of 100 units is placed, either on YES or NO. Let's consider both cases:

   - **Bet on YES**: The number of YES shares increases by 100, so the new number of YES shares is $y + 100$, while the number of NO shares remains $n$.
   - **Bet on NO**: The number of NO shares increases by 100, so the new number of NO shares is $n + 100$, while the number of YES shares remains $y$.

3. **New Parameter** $p_{\text{new}}$:
   
   The new parameter $p_{\text{new}}$ must be adjusted to maintain the same probability $P_{YES}$. We solve for $p_{\text{new}}$ such that:

   $$
   \frac{p_{\text{new}} \cdot (n + \Delta n)}{p_{\text{new}} \cdot (n + \Delta n) + (1 - p_{\text{new}}) \cdot (y + \Delta y)} = P_{YES}
   $$

   Here:

   - $\Delta y$ and $\Delta n$ represent changes in the YES and NO shares due to the bet (e.g., 100).
   - $P_{YES}$ is the probability before the bet.

4. **Solving for** $p_{\text{new}}$:

   Rearranging the equation gives:
   $$
   p_{\text{new}} \cdot (n + \Delta n) = P_{YES} \cdot \left( p_{\text{new}} \cdot (n + \Delta n) + (1 - p_{\text{new}}) \cdot (y + \Delta y) \right)
   $$

   This is a nonlinear equation in $p_{\text{new}}$, which can be solved numerically. After solving for $p_{\text{new}}$, we can calculate the new market probabilities.

### **Change in Liquidity for a Trader**

For a trader who adds liquidity to the pool (with capital $l$), we calculate the change in liquidity as:

$$
s_{l,t} = \Delta \text{liquidity} = (y_t + l)^p \cdot (n_t + l)^{p-1} - y_t^p \cdot n_t^{p-1}
$$

Where:

- $y_t$ and $n_t$ are the YES and NO shares at the time of liquidity provision.
- $l$ is the capital (or liquidity) added by the trader.
- $p$ is the current market parameter.

The trader then receives YES and NO shares based on their proportion of the total liquidity:

$$
\frac{s}{\sum s_{l,t}} \cdot y
$$

for YES shares, and similarly for NO shares.

### **Fees and Their Impact on the Pool**

Manifold charges fees on each trade:

- The fee is given by $\text{fee} = 13\% \times (1 - \text{post-bet probability}) \times \text{trade amount}$.
- Of this, 6% goes to the liquidity pool, 6% goes to the market creator, and 1% goes to Manifold.

These fees are converted into equal numbers of YES and NO shares and added to the pool. The new liquidity is calculated after updating the parameter $p_{\text{new}}$. Liquidity providers can then earn these fees when redeeming their shares.

### **Maximum Payout Calculation**

The maximum payout a trader can receive depends on the probability after the bet. For example:

- If the trader bets on YES, their maximum payout (if YES wins) is proportional to the number of YES shares they bought, and is calculated as:

$$
\text{Payout}_{YES} = \frac{\text{Bet Amount}}{P_{YES, \text{new}}}
$$

- If the trader bets on NO, the payout is:

$$
\text{Payout}_{NO} = \frac{\text{Bet Amount}}{P_{NO, \text{new}}}
$$

Where $P_{YES, \text{new}}$ and $P_{NO, \text{new}}$ are the updated probabilities after the bet.

### **Steps to Calculate the Change in Probability After the Bet**

To summarize the process of calculating the change in probability after a bet:

1. **Determine the Initial Market Conditions**:

   - Use the current quantities of YES ($y$) and NO ($n$) shares and the parameter $p$ to calculate the initial market probability $P_{YES}$.

2. **Apply the Bet**:

   - Adjust the number of YES or NO shares based on the bet amount.

3. __Solve for the New $p_{\text{new}}$__:

   - Use the equation for market probability to solve for the new $p_{\text{new}}$, keeping the market probability constant.

4. **Calculate the New Probabilities**:

   - Once $p_{\text{new}}$ is determined, compute the new market probabilities for YES and NO.

5. **Determine the Maximum Payout**:

   - Calculate the maximum payout based on the updated probabilities.

### Conclusion

To compute the exact change in probability and payout, you would follow these steps using specific input values for $y$, $n$, $p$, and the bet size. If you provide specific numbers, I can walk through the detailed computation with you.

In the context of a prediction market like Manifold’s modified CPMM, **log-odds** and **elasticity** provide insight into how the market probability shifts when large amounts of currency are traded on a particular outcome. Elasticity measures the sensitivity of the market price to changes in the number of YES/NO shares caused by a bet. Let’s break down both concepts.

### **Log-Odds and Market Probability**

The **log-odds** of an event happening (YES in this case) is a transformation of the market probability into a logarithmic scale. This scale is useful because probabilities are bounded between 0 and 1, while log-odds can take on values from $-\infty$ to $+\infty$, allowing for easier analysis of large movements in market prices.

For the probability $P_{YES}$ of a YES share, the __log-odds__ is defined as:

$$
\text{Log-Odds}_{YES} = \ln\left(\frac{P_{YES}}{1 - P_{YES}}\right)
$$

Where:

- $P_{YES}$ is the market probability of YES occurring.
- $1 - P_{YES}$ is the market probability of NO occurring.

#### **Example**:

- If $P_{YES} = 0.6$, the log-odds is:
   $$
   \text{Log-Odds}_{YES} = \ln\left(\frac{0.6}{1 - 0.6}\right) = \ln\left(\frac{0.6}{0.4}\right) = \ln(1.5) \approx 0.405
   $$

A **positive log-odds** means the probability of YES is greater than 50%, while a **negative log-odds** indicates that NO is more likely.

### **Elasticity and its Role in Price Movements**

**Elasticity** measures how sensitive the market’s log-odds are to changes in the number of YES or NO shares in the pool. In Manifold’s CPMM model, **elasticity** refers to the **log-odds change** for a given amount of currency (e.g., a 10,000-unit trade on YES or NO).

Formally, elasticity $E$ is the change in log-odds per unit of trade (currency) or liquidity added to the pool.

#### **Elasticity and Log-Odds Change**:

Suppose you make a bet of size $B$ on YES. The resulting **change in log-odds** (denoted as $\Delta \text{Log-Odds}$) is related to elasticity and the amount bet. For a bet of $B$ currency units on YES, the change in log-odds is:

$$
\Delta \text{Log-Odds}_{YES} = E \times \ln\left(1 + \frac{B}{k}\right)
$$

Where:

- $E$ is the elasticity.
- $B$ is the amount bet.
- $k$ is the current liquidity in the market or a scaling factor representing the constant in the CPMM equation.

### **Interpreting Elasticity**

- **High Elasticity** means the market is highly sensitive to bets, meaning even a small bet will cause a significant shift in the log-odds (and hence, the probability).
- **Low Elasticity** means the market is more resilient to price changes from individual bets, so larger bets are needed to significantly change the market probability.

In practical terms, **elasticity** helps you understand how a large bet (e.g., 10,000 units) would affect the market probability. A **higher elasticity** means a bigger shift in probability for a given bet size.

### **Log-Odds Change from a 10,000-Unit Trade on YES or NO**

Let’s calculate the log-odds change for a 10,000-unit bet, assuming you know the current **elasticity** $E$ and **liquidity** $k$.

For a bet of $B = 10,000$ units:

$$
\Delta \text{Log-Odds}_{YES} = E \times \ln\left(1 + \frac{10,000}{k}\right)
$$

- If $E$ is high, this will result in a large increase in log-odds, meaning the probability will shift significantly.
- If $E$ is low, the impact on the log-odds and therefore the probability will be smaller.

### **Step-by-Step Calculation Example**:

Suppose:

- Elasticity $E = 0.02$,
- Current liquidity $k = 50,000$.

For a 10,000-unit trade on YES:

$$
\Delta \text{Log-Odds}_{YES} = 0.02 \times \ln\left(1 + \frac{10,000}{50,000}\right) = 0.02 \times \ln(1.2) \approx 0.02 \times 0.182 = 0.00364
$$

This means the log-odds increase by 0.00364 for a 10,000-unit bet. You can then convert this back to the new market probability.

### **Converting Log-Odds Back to Probability**

Once you know the change in log-odds, you can convert it back into the new probability $P_{YES,\text{new}}$:

$$
P_{YES, \text{new}} = \frac{e^{\text{Log-Odds}_{YES,\text{new}}}}{1 + e^{\text{Log-Odds}_{YES,\text{new}}}}
$$

Where $\text{Log-Odds}_{YES,\text{new}}$ is the initial log-odds plus the change $\Delta \text{Log-Odds}_{YES}$.

### **Summary**

- **Log-odds**: Represents the probability of an event happening in logarithmic form, allowing easier analysis of market changes.
- **Elasticity**: Indicates how sensitive the market is to a bet. It measures the change in log-odds per unit of trade. Higher elasticity means a larger shift in probability for a given bet size.
- **Log-Odds Change for a 10,000-Unit Bet**: Is calculated using elasticity and the current liquidity, helping you understand how much the market probability will change when a large bet is placed.

If you have specific values for elasticity and liquidity, I can help you calculate the exact change in probability for a 10,000-unit bet!