# Slide 1
Identifying fraudulent transactions in e-commerce

Catching the bots (and the sneaky humans)

Presented by Bravin and Chidera

# Slide 2
## 1. Problem Definition
E-commerce platforms lose money to fraudulent orders placed with stolen payment credentials.

a chargeback typically costs the merchant the goods, the fulfilment and shipping, and a processing fee, so the realised loss on an undetected fraudulent order commonly exceeds twice its face value.

This project aims to build a prediction model that will 
Estimate the probability that a new user's first transaction is fraudulent, using only information available at the instant of purchase.

Then based on the probability decide whether
a newly registered user can complete the order normally, require a step-up verification such as an SMS code, or hold the order for manual review?

# Slide 3
## Dataset Exploration
The project uses the Fraud E-commerce dataset published on Kaggle (vbinh002/fraud-ecommerce).

Fraud_Data.csv — 151,112 rows and 11 columns, one row per user, describing that user's first transaction:

| Variable | Description |
|---|---|
| `user_id` | Unique user identifier |
| `signup_time` | Account creation timestamp (GMT) |
| `purchase_time` | Purchase timestamp (GMT) |
| `purchase_value` | Item cost (USD) |
| `device_id` | Device identifier, unique per physical device |
| `source` | Acquisition channel: Ads, SEO, or Direct |
| `browser` | Browser used |
| `sex`, `age` | User demographics |
| `ip_address` | Numeric IP address |
| `class` | Target: 1 = fraudulent |

# Slide 4

Here is the top 5 rows of the dataset

| # | signup_time | purchase_time | purchase_value | device_id | source | browser | sex | age | ip_address | class |
|---|---:|---|---:|---|---|---|---|---:|---:|---:|
| 0 | 22058 | 2015-02-24 22:55:49 | 34 | QVPSPJUOCKZAR | SEO | Chrome | M | 39 | 7.327584e+08 | 0 |
| 1 | 333320 | 2015-06-07 20:39:50 | 16 | EOGFQPIZPYXFZ | Ads | Chrome | F | 53 | 3.503114e+08 | 0 |
| 2 | 1359 | 2015-01-01 18:52:44 | 15 | YSSKYOSJHPPLJ | SEO | Opera | M | 53 | 2.621474e+09 | 1 |
| 3 | 150084 | 2015-04-28 21:13:25 | 44 | ATGTXKYKUDUQN | SEO | Safari | M | 41 | 3.840542e+09 | 0 |
| 4 | 221365 | 2015-07-21 07:09:52 | 39 | NAUITBZFJKHWW | Ads | Safari | M | 45 | 4.155831e+08 | 0 |

# Slide 5

IpAddress_to_Country.csv — 138,846 rows mapping numeric IP ranges (lower_bound_ip_address, upper_bound_ip_address) to a country, joined by interval lookup rather than equality.


Here is the top 5 rows of the dataset

| # | lower_bound_ip_address | upper_bound_ip_address | country |
|---|---:|---:|---|
| 0 | 16777216 | 16777471 | Australia |
| 1 | 16777472 | 16777727 | China |
| 2 | 16777728 | 16778239 | China |
| 3 | 16778240 | 16779263 | Australia |
| 4 | 16779264 | 16781311 | China |

# Slide 6
## Data Quality 
There are zero nulls, zero duplicate rows and zero duplicate user IDs.

## Quality Checks Done

| Issue | Finding | Action |
|---|---|---|
| Missing values | 0 across all 14 columns in both files | none required |
| Duplicate rows | 0 full duplicates, 0 duplicate `user_id` | none required |
| Timestamp types | `signup_time` and `purchase_time` read as strings | cast to `datetime64` |
| Negative durations | 0 purchases precede their signup | none required |
| `ip_address` type | stored as float with a decimal component | cast to `int64` for range matching |
| IP range join | sorted, non-overlapping, monotonic bounds | `searchsorted` interval join |
| Unmatched IPs | 21,966 rows, 14.54%, fall outside every range | labelled `Unknown`, retained |
| Outliers | `purchase_value` max 154 against a median of 35 | retained, plausible basket values |

## Class Balance

| Class | Count | Share |
|---|---:|---:|
| Legitimate | 136,961 | 90.64% |
| Fraudulent | 14,151 | 9.36% |

# Slide 7
The IP to Country Join
## Volume by Country

| Country | Transactions |
|---|---:|
| United States | 58,049 |
| Unknown | 21,966 |
| China | 12,038 |
| Japan | 7,306 |
| United Kingdom | 4,490 |
| Korea, Republic of | 4,162 |
| Germany | 3,646 |
| France | 3,161 |
| Canada | 2,975 |
| Brazil | 2,961 |

## Fraud Rate by Country
Countries with at least 500 transactions, ranked by fraud rate. Base rate is 9.36%.

| Country | Transactions | Fraud rate |
|---|---:|---:|
| Norway | 609 | 13.0% |
| Mexico | 1,121 | 12.8% |
| Sweden | 1,090 | 12.0% |
| Canada | 2,975 | 11.7% |
| India | 1,310 | 11.5% |
| United Kingdom | 4,490 | 10.6% |
| Argentina | 661 | 10.0% |
| Japan | 7,306 | 9.8% |
| United States | 58,049 | 9.6% |
| France | 3,161 | 9.5% |

## Slide 8
## Data Analysis

### Time Between Signup and Purchase


Subtracting signup_time from purchase_time produces the single most informative variable in the dataset.

Legitimate users take weeks or months to make a first purchase, with a median gap of 60 days.

A large block of transactions instead completes in exactly one second.

### Fraud Rate by Signup to Purchase Gap
| Gap | Transactions | Fraud rate |
|---|---:|---:|
| 1 second or less | 7,600 | 100.0% |
| 1 minute to 1 hour | 41 | 9.8% |
| 1 hour to 1 day | 1,117 | 3.9% |
| 1 to 7 days | 7,082 | 4.5% |
| 7 to 30 days | 27,629 | 4.4% |
| Over 30 days | 107,643 | 4.6% |

#### What the One-Second Block Means
This is a bot signature, not human behaviour.

7,600 accounts signed up and purchased within the same second, and all 7,600 are labelled fraud. No legitimate transaction appears in that bucket at all.

It is a perfect separator over 5% of the data, and it accounts for 53.7% of every fraud case in the dataset.

### Shared Devices

device_id is near-unique for most users, but a minority repeat. Fraud rate rises sharply with the number of accounts behind a single device.

| Accounts per device | Transactions | Fraud rate |
|---|---:|---:|
| 1 | 131,781 | 3.0% |
| 2 | 10,654 | 22.9% |
| 3 | 270 | 24.4% |
| 4 | 16 | 62.5% |
| 5 | 65 | 80.0% |
| 6 or more | 8,326 | 91.0% |

The reuse distribution is bimodal: normal single use, then a separate cluster of 741 devices carrying 6 to 20 accounts each.

# Slide 8
Shared IP Addresses

The same pattern appears on IP addresses, which is expected if the same automation drives both.

| Accounts per IP | Transactions | Fraud rate |
|---|---:|---:|
| 1 | 142,748 | 4.6% |
| 2 | 6 | 16.7% |
| 3 | 6 | 66.7% |
| 4 | 16 | 81.3% |
| 5 | 65 | 80.0% |
| 6 or more | 8,271 | 91.5% |
