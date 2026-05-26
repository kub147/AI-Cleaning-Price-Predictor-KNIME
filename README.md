# 🧹 AI Cleaning Price Predictor
### Predictive Modeling & Agentic Workflow Project for Cleaning Services
**EIACD Project · FCUP 2025/2026**  
**Authors:** Aleksandra Rodzinka · Jakub Wilk · Łukasz Furmanek

---

## 📌 Project Overview

An agentic AI system built in **KNIME Analytics Platform** that acts as a virtual cleaning service assistant. The agent converses with the customer, collects information about their home, and uses a trained Machine Learning model to predict the cost of cleaning in EUR.

**Example conversation:**
> 🧑 *"Hi, I have a 120m² apartment, 3 rooms, 1 bathroom, I have a cat, no garden, biweekly cleaning"*  
> 🤖 *"The estimated cost is €67. The price range is between €60 and €74. The main factors affecting your price are home size, cleaning frequency, and having a pet."*

---

## 🗂️ Repository Structure

```
/
├── Tools/
│   ├── tool1_predict.knwf           # Tool 1: Price prediction
│   ├── tool2_confidence.knwf        # Tool 2: Confidence interval
│   ├── tool3_text_to_table.knwf     # Tool 3: Text → Table via LLM
│   └── tool4_feature_importance.knwf # Tool 4: Feature importance (bonus)
├── agent_main.knwf                  # Main agent workflow
├── model_creation.knwf              # Model training & saving
├── cleaning_dataset_eur.csv         # Dataset (1,000 rows)
└── README.md
```

---

## 🗃️ Dataset

**File:** `cleaning_dataset_eur.csv` | **1,000 rows** | **7 columns** | Synthetic data

| Column | Type | Description | Range |
|---|---|---|---|
| `area_m2` | Integer | Home size in square meters | 30–200 |
| `num_rooms` | Integer | Number of rooms | 1–8 |
| `num_bathrooms` | Integer | Number of bathrooms | 1–3 |
| `has_pets` | Integer (0/1) | Whether the home has pets | 0 or 1 |
| `has_garden` | Integer (0/1) | Whether there is a garden | 0 or 1 |
| `cleaning_frequency` | String | How often cleaning is done | weekly / biweekly / monthly |
| `cleaning_price_eur` | Float | **Target: cleaning cost in EUR** | €18–€112 |

**Price formula used to generate data:**
```
price = (20 + area_m2×0.22 + bathrooms×8 + pets×12 + garden×15 + noise) × frequency_multiplier
```
Where: weekly=1.0, biweekly=0.85, monthly=0.7

---

## ⚙️ Requirements

- **KNIME Analytics Platform** v5.11+
- **KNIME AI Extension** v5.8+
- **Groq API key** (free at https://console.groq.com)
- Model: `llama-3.3-70b-versatile` via Groq

---

## 🚀 How to Run

### Step 1 – Train and save the model
1. Open `model_creation.knwf` in KNIME
2. Execute all nodes
3. The trained model is saved automatically via **Model Writer**

### Step 2 – Configure Groq API key
In `tool3_text_to_table.knwf` and `agent_main.knwf`:
1. Open the **Credentials Configuration** node
2. Enter your Groq API key in the **password** field
3. Save

### Step 3 – Run the agent
1. Open `agent_main.knwf`
2. Execute all nodes
3. Open **Agent Chat View** → start chatting!

---

## 🔧 Detailed Workflow Descriptions

---

### 📦 model_creation.knwf — Model Training

**Purpose:** Load the dataset, train a Linear Regression model, evaluate it, and save it to disk for use by the tools.

**Node flow:**
```
CSV Reader → One to Many → Table Partitioner → Linear Regression Learner → Regression Predictor → Numeric Scorer
                                                         ↓
                                                   Model Writer
```

**Node-by-node explanation:**

| Node | Configuration | Purpose |
|---|---|---|
| **CSV Reader** | Load `cleaning_dataset_eur.csv` | Reads the training dataset |
| **One to Many** | Column: `cleaning_frequency` | Encodes text (weekly/biweekly/monthly) into binary columns (0/1) |
| **Table Partitioner** | 80% train / 20% test, Random | Splits data for training and evaluation |
| **Linear Regression Learner** | Target: `cleaning_price_eur`, includes all other columns | Trains the regression model |
| **Regression Predictor** | Left port ← model, Right port ← test data | Makes predictions on unseen test data |
| **Numeric Scorer** | First col: `cleaning_price_eur`, Second col: `Prediction (cleaning_price_eur)` | Evaluates model performance |
| **Model Writer** | Connected to top port of Learner | Saves the trained model to file |

**Model results:**
| Metric | Value |
|---|---|
| R² | **0.913** |
| Mean Absolute Error | 3.68 EUR |
| Root Mean Squared Error | 4.68 EUR |
| Mean Absolute % Error | 6.9% |

---

### 🔧 Tool 1 — predict_price (`tool1_predict.knwf`)

**Purpose:** Receives a table with home features as input and returns the predicted cleaning price in EUR using the pre-trained model.

**Node flow:**
```
Container Input (Table) → One to Many → Model Reader → Regression Predictor → Container Output (Table)
```

**Node-by-node explanation:**

| Node | Configuration | Purpose |
|---|---|---|
| **Container Input (Table)** | Name: `new_data` | Receives home features as a table from the agent |
| **One to Many** | Column: `cleaning_frequency` | Encodes frequency column same as during training |
| **Model Reader** | Path to saved model file | Loads the trained Linear Regression model |
| **Regression Predictor** | Left port ← Model Reader, Right port ← One to Many | Applies the model to input data |
| **Container Output (Table)** | — | Returns table with `Prediction (cleaning_price_eur)` column |

**Input table format:**
```
area_m2 | num_rooms | num_bathrooms | has_pets | has_garden | cleaning_frequency
120     | 3         | 1             | 1        | 0          | biweekly
```

**Output:**
```
Prediction (cleaning_price_eur)
57.34
```

---

### 🔧 Tool 2 — get_confidence (`tool2_confidence.knwf`)

**Purpose:** Extends the prediction with a confidence interval of ±10%, giving the customer a realistic price range.

**Node flow:**
```
Container Input (Table) → One to Many → Model Reader → Regression Predictor → Math Formula → Math Formula → Container Output (Table)
```

**Node-by-node explanation:**

| Node | Configuration | Purpose |
|---|---|---|
| **Container Input (Table)** | Name: `new_data` | Receives home features from the agent |
| **One to Many** | Column: `cleaning_frequency` | Encodes frequency column |
| **Model Reader** | Path to saved model file | Loads the trained model |
| **Regression Predictor** | Left ← model, Right ← data | Generates price prediction |
| **Math Formula #1** | `$Prediction (cleaning_price_eur)$ * 0.90` → `confidence_low` | Lower bound of price range |
| **Math Formula #2** | `$Prediction (cleaning_price_eur)$ * 1.10` → `confidence_high` | Upper bound of price range |
| **Container Output (Table)** | — | Returns prediction + confidence_low + confidence_high |

**Output example:**
```
Prediction | confidence_low | confidence_high
67.40      | 60.66          | 74.14
```

---

### 🔧 Tool 3 — text_to_table (`tool3_text_to_table.knwf`)

**Purpose:** The most important tool. Accepts a plain-text description of a home from the customer and uses an LLM (Llama 3.3 via Groq) to extract structured features and return them as a table ready for prediction.

**Node flow:**
```
Credentials Configuration → OpenAI Authenticator → OpenAI LLM Selector
                                                           ↓
Container Input (Variable) → Table Creator → LLM Prompter → String to JSON → JSON Path → Ungroup → Column Filter → Container Output (Table)
```

**Node-by-node explanation:**

| Node | Configuration | Purpose |
|---|---|---|
| **Credentials Configuration** | Password: Groq API key | Stores API credentials securely |
| **OpenAI Authenticator** | Advanced settings → Base URL: `https://api.groq.com/openai/v1` | Connects to Groq using OpenAI-compatible API |
| **OpenAI LLM Selector** | Manual → `llama-3.3-70b-versatile` | Selects the LLM model |
| **Container Input (Variable)** | Name: `user_description` | Receives customer's text description |
| **Table Creator** | Column: `prompt`, one row with the customer's text | Creates input table for LLM |
| **LLM Prompter** | System message (see below), Prompt column: `prompt`, Output format: JSON, Response column: `response` | Sends text to LLM and gets JSON back |
| **String to JSON** | Input: `response` | Converts LLM text response to JSON type |
| **JSON Path** | 6 paths extracting each field (see below) | Extracts individual values from JSON |
| **Ungroup** | All 6 columns | Converts list values to scalar values |
| **Column Filter** | Keep only the 6 feature columns | Removes `prompt` and `response` columns |
| **Container Output (Table)** | — | Returns structured table ready for Tool 1 |

**LLM System Message:**
```
You are a data extraction assistant. Extract home cleaning information 
from the user's text and return ONLY a valid JSON object with these exact fields:
- area_m2 (integer)
- num_rooms (integer)
- num_bathrooms (integer)
- has_pets (0 or 1)
- has_garden (0 or 1)
- cleaning_frequency (must be exactly: "weekly", "biweekly", or "monthly")

Return ONLY the JSON, no explanation, no markdown, no backticks.
```

**JSON Path expressions:**

| Output Column | JSON Path |
|---|---|
| `area_m2` | `$.area_m2` |
| `num_rooms` | `$.num_rooms` |
| `num_bathrooms` | `$.num_bathrooms` |
| `has_pets` | `$.has_pets` |
| `has_garden` | `$.has_garden` |
| `cleaning_frequency` | `$.cleaning_frequency` |

**Example:**
- Input: *"120m² apartment, 3 rooms, 1 bathroom, have a cat, no garden, biweekly"*
- LLM returns: `{"area_m2": 120, "num_rooms": 3, "num_bathrooms": 1, "has_pets": 1, "has_garden": 0, "cleaning_frequency": "biweekly"}`
- Output table: ready for Tool 1

---

### 🔧 Tool 4 — feature_importance (`tool4_feature_importance.knwf`) ⭐ Bonus

**Purpose:** Returns a table explaining which home features have the most impact on the cleaning price, helping the customer understand what drives the cost.

**Node flow:**
```
Table Creator → Container Output (Table)
```

**Output table:**

| Feature | Importance | Impact |
|---|---|---|
| area_m2 | High | Largest driver of price |
| num_bathrooms | Medium | Each bathroom adds ~€8 |
| has_garden | Medium | Garden adds ~€15 |
| has_pets | Low-Medium | Pets add ~€12 |
| cleaning_frequency | Medium | Weekly costs more than monthly |
| num_rooms | Low | Correlated with area |

> **Note:** Importance values are derived from the Linear Regression model coefficients trained on the dataset (R²=0.913). The area_m2 coefficient is 0.22, meaning each additional m² adds approximately €0.22 to the base price.

---

### 🤖 agent_main.knwf — The AI Orchestrator

**Purpose:** The main workflow that ties all tools together into a conversational AI agent. The agent receives user messages, decides which tools to call, executes them, and returns a human-readable response.

**Node flow:**
```
Credentials Configuration → OpenAI Authenticator → OpenAI LLM Selector ──────────────┐
                                                                                       ↓
List Files/Folders → Workflow to Tool ────────────────────────────────────→ Agent Prompter → Agent Chat View
```

**Node-by-node explanation:**

| Node | Configuration | Purpose |
|---|---|---|
| **Credentials Configuration** | Password: Groq API key | API credentials for the agent's LLM |
| **OpenAI Authenticator** | Base URL: `https://api.groq.com/openai/v1` | Groq connection |
| **OpenAI LLM Selector** | Manual → `llama-3.3-70b-versatile` | LLM that powers the agent |
| **List Files/Folders** | Path to `Tools/` folder, extension: `.knwf` | Discovers all available tools |
| **Workflow to Tool** | Reads all 4 tools from folder | Registers tools so the agent can call them |
| **Agent Prompter** | System message (see below) | Orchestrates the conversation and tool calls |
| **Agent Chat View** | — | Opens interactive chat window |

**Agent System Message:**
```
You are a friendly cleaning service assistant. 
Help the customer estimate the cost of cleaning their home in EUR.
Ask them one question at a time about:
1. Home size in square meters
2. Number of rooms
3. Number of bathrooms
4. Whether they have pets (yes/no)
5. Whether they have a garden (yes/no)
6. Preferred cleaning frequency (weekly, biweekly, or monthly)

After collecting all information, use text_to_table to convert their 
answers, then predict_price to estimate the cost, then get_confidence 
to show the price range, and feature_importance to explain what affects 
the price.

Always respond in the same language as the customer.
After collecting all 6 answers, immediately call all tools without 
waiting for user confirmation.
Always display prices in EUR (€).
```

**How the agent works internally:**
1. User sends a message in the chat
2. Agent Prompter sends it to the LLM with the system message and list of available tools
3. LLM decides which tool(s) to call based on the conversation
4. KNIME executes the selected tool workflow(s)
5. Tool output is sent back to the LLM
6. LLM formulates a human-readable response
7. Response is displayed in Agent Chat View

---

## 📊 Full System Architecture

```
Customer Message
      ↓
Agent Prompter (LLM: Llama 3.3-70b)
      ↓
┌─────────────────────────────────────────┐
│ Selects and calls tools as needed:      │
│                                         │
│  text_to_table  →  structured table     │
│      ↓                                  │
│  predict_price  →  price in EUR         │
│      ↓                                  │
│  get_confidence →  price range ±10%     │
│      ↓                                  │
│  feature_importance → what drives cost  │
└─────────────────────────────────────────┘
      ↓
Human-readable response to customer
```

---

## 📈 Model Performance Summary

| Metric | Value | Interpretation |
|---|---|---|
| **R²** | 0.913 | Model explains 91.3% of price variation |
| **MAE** | 3.68 EUR | Average prediction error of ~3.68 EUR |
| **RMSE** | 4.68 EUR | Slightly penalises larger errors |
| **MAPE** | 6.9% | Average percentage error under 7% |

The high R² is expected given that the dataset was generated using a linear formula — Linear Regression is the ideal model for this data.

---

## 🔑 Important Notes

1. **One to Many encoding** must be applied consistently — both during training and in every tool that uses the model. The `cleaning_frequency` column must be encoded before passing data to the Regression Predictor.

2. **Column names must match exactly** between Tool 3 output and Tool 1 input: `area_m2`, `num_rooms`, `num_bathrooms`, `has_pets`, `has_garden`, `cleaning_frequency`.

3. **agent_main.knwf must be in a separate folder** from the tools. If it is in the same folder as the tools, List Files/Folders will pick it up and the agent will try to call itself, causing an infinite loop.

4. **Each team member needs their own Groq API key.** The free tier has daily token limits. Replace the key in Credentials Configuration if you hit the limit.

5. **The model file** must be present on disk before running any tool. Run `model_creation.knwf` first.