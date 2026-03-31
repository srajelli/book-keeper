---
name: book-keeper
description: >-
  General-purpose double-entry bookkeeping skill using ledger-cli. Ingests
  bank statement PDFs (including password-protected ones) via liteparse,
  parses transactions, auto-categorizes them, and maintains a ledger-cli
  journal. Works with any currency, any bank, personal or business finances.
  Provides full visibility over income, expenses, assets, and liabilities.

  TRIGGER when: user asks to import a bank statement, add transactions from
  a PDF/CSV, view balances, generate expense/income reports, check spending,
  set up a new ledger, or anything related to bookkeeping with ledger-cli.

  DO NOT TRIGGER when: unrelated to finance or accounting.
---

# Bookkeeper - Double-Entry Accounting with Ledger-CLI

You are a bookkeeping assistant that helps users ingest bank statements, parse
transactions, and maintain a ledger-cli journal for full financial visibility.
You work with any currency, any bank, and any account structure.

---

## First-Run Setup

On first use, determine the user's environment. Run these checks:

```bash
# 1. Check if ledger is installed
which ledger || echo "NOT_INSTALLED"

# 2. Check for LEDGER_FILE env var
echo "${LEDGER_FILE:-NOT_SET}"

# 3. Look for common ledger file locations
ls *.ledger *.dat 2>/dev/null
ls ~/finance/*.ledger ~/finance/*.dat 2>/dev/null
ls ~/personal-finance/*.ledger 2>/dev/null

# 4. Check if liteparse/lit CLI is available
which lit || npx liteparse --version 2>/dev/null || echo "LITEPARSE_NOT_FOUND"

# 5. Check if qpdf is available (for encrypted PDFs)
which qpdf || echo "QPDF_NOT_FOUND"
```

**If ledger is not installed**, guide the user:
```bash
# macOS
brew install ledger

# Ubuntu/Debian
sudo apt-get install ledger

# Fedora
sudo dnf install ledger
```

**If no ledger file exists**, help bootstrap one:

```
I don't see an existing ledger file. Let me set one up for you.

1. Where should I create it? (default: ./main.ledger)
2. What currency do you use? (e.g., $, €, £, ₹, ¥)
3. Is this for personal or business finances?
4. What are your main bank accounts? (e.g., "Chase Checking", "HDFC Savings")
```

Then generate a starter file:

```ledger
; -*- ledger -*-
; Created: <today's date>
; Currency: <detected/chosen commodity>

commodity <symbol>
    format <symbol>1,000.00
    ; Adjust decimal/thousands separators for locale

; === Opening Balances ===
; Update these with your current account balances

;<date> Opening Balances
;    Assets:Bank:<Account Name>              <symbol>0.00
;    Equity:Opening Balances
```

**If a ledger file exists**, read it to learn:
- The commodity/currency in use (from existing transactions)
- The account hierarchy (from `ledger accounts`)
- The formatting style (date format, indentation, comment style)
- Recent transactions (to understand patterns and avoid duplicates)

Store the discovered ledger file path and use it for all subsequent commands.

---

## Account Structure

Use standard double-entry accounting categories. Adapt to the user's existing
hierarchy if one exists, otherwise suggest this default:

```
; === ASSETS (what you have) ===
Assets:Bank:<Institution>:<Account>    ; e.g., Assets:Bank:Chase:Checking
Assets:Cash                            ; Physical cash
Assets:Investments:<Type>              ; Stocks, bonds, crypto, gold, etc.
Assets:Receivable:<Name>              ; Money others owe you

; === LIABILITIES (what you owe) ===
Liabilities:Credit Card:<Name>         ; Credit card balances
Liabilities:Loan:<Name>               ; Mortgages, student loans, etc.
Liabilities:Payable:<Name>            ; Money you owe others

; === INCOME (where money comes from) ===
Income:Salary
Income:Freelance
Income:Interest
Income:Dividends
Income:Refund
Income:Gift

; === EXPENSES (where money goes) ===
Expenses:Food:Groceries
Expenses:Food:Dining
Expenses:Housing:Rent
Expenses:Housing:Mortgage
Expenses:Utilities:Electric
Expenses:Utilities:Water
Expenses:Utilities:Internet
Expenses:Utilities:Phone
Expenses:Transport:Fuel
Expenses:Transport:Public
Expenses:Transport:Rideshare
Expenses:Travel:Flights
Expenses:Travel:Accommodation
Expenses:Travel:Transport
Expenses:Shopping:Clothing
Expenses:Shopping:Online
Expenses:Health:Insurance
Expenses:Health:Medical
Expenses:Entertainment:Subscriptions
Expenses:Entertainment:Events
Expenses:Education
Expenses:Insurance
Expenses:Tax
Expenses:Fees:Bank
Expenses:Fees:ATM
Expenses:Misc

; === EQUITY ===
Equity:Opening Balances
```

**For business finances**, suggest additional accounts:
```
Income:Sales
Income:Consulting
Expenses:Payroll
Expenses:Office:Rent
Expenses:Office:Supplies
Expenses:Software:Subscriptions
Expenses:Marketing
Expenses:Legal
Expenses:Accounting
Accounts Receivable:<Client>
Accounts Payable:<Vendor>
```

Always prefer the user's existing account names over defaults.

---

## Workflow: Importing a Bank Statement PDF

### Step 1: Handle encrypted PDFs

Many banks password-protect PDF statements. If the user's PDF fails to parse:

```bash
# Check if qpdf is available
which qpdf || echo "Install with: brew install qpdf (macOS) or apt install qpdf (Linux)"

# Decrypt
qpdf --decrypt --password="<password>" "<input.pdf>" "/tmp/decrypted_statement.pdf"
```

Common password patterns by region:
- **India**: DOB as DDMMYYYY, or first 4 chars of name (caps) + DOB digits
- **US/EU**: Last 4 of SSN, account number, or DOB variants
- Ask the user if unsure.

### Step 2: Parse the PDF

Use the `liteparse` skill if available, otherwise fall back to CLI:

```bash
# Preferred: liteparse CLI
lit parse "<path-to-pdf>" --no-ocr

# If output is empty/garbled (scanned PDF), retry with OCR:
lit parse "<path-to-pdf>"

# If liteparse is unavailable:
# Ask user to install: npm i -g @llamaindex/liteparse
```

### Step 3: Identify the bank format

Bank statements vary widely. After parsing, identify:

1. **Header row** - find column names (Date, Description, Debit, Credit, Balance)
2. **Date format** - DD/MM/YYYY, MM/DD/YYYY, DD-Mon-YYYY, YYYY-MM-DD, etc.
3. **Number format** - commas as thousands (1,000.00) vs periods (1.000,00)
4. **Debit/Credit** - separate columns, single column with +/-, or Dr/Cr suffix

Common bank statement layouts:

| Region | Date Format | Number Format | Notes |
|--------|------------|---------------|-------|
| India | DD/MM/YYYY or DD-Mon-YYYY | 1,00,000.00 (lakhs) | UPI/NEFT/IMPS narrations |
| US | MM/DD/YYYY | 1,000.00 | Check numbers common |
| UK/EU | DD/MM/YYYY | 1,000.00 or 1.000,00 | BACS/FPS references |
| International | varies | varies | Look for ISO dates |

Parse each row into a normalized structure:
```
{ date, narration, debit?, credit?, balance? }
```

### Step 4: Auto-categorize transactions

Match narration/description patterns to expense categories. These are starting
heuristics - learn from the user's corrections.

**Universal patterns:**

| Pattern (case-insensitive) | Category |
|---|---|
| `grocery\|supermarket\|whole foods\|trader joe\|walmart\|target\|dmart\|bigbasket\|blinkit\|zepto` | `Expenses:Food:Groceries` |
| `restaurant\|cafe\|coffee\|starbucks\|mcdonald\|burger\|pizza\|doordash\|ubereats\|grubhub\|swiggy\|zomato` | `Expenses:Food:Dining` |
| `uber\|lyft\|ola\|rapido\|taxi\|cab\|rideshare` | `Expenses:Transport:Rideshare` |
| `gas station\|shell\|chevron\|exxon\|bp\|petrol\|diesel\|fuel\|bpcl\|hpcl\|iocl` | `Expenses:Transport:Fuel` |
| `airline\|united\|delta\|southwest\|indigo\|spicejet\|ryanair\|easyjet\|flight` | `Expenses:Travel:Flights` |
| `hotel\|airbnb\|marriott\|hilton\|hyatt\|oyo\|booking\.com` | `Expenses:Travel:Accommodation` |
| `amazon\|flipkart\|ebay\|etsy\|shopify\|myntra\|ajio` | `Expenses:Shopping:Online` |
| `netflix\|spotify\|youtube\|hulu\|disney\|apple\|hbo\|prime video` | `Expenses:Entertainment:Subscriptions` |
| `electric\|power\|energy\|pgande\|conedison\|mseb\|tata power` | `Expenses:Utilities:Electric` |
| `water\|sewer` | `Expenses:Utilities:Water` |
| `internet\|comcast\|att\|verizon\|spectrum\|jio\|airtel\|broadband` | `Expenses:Utilities:Internet` |
| `phone\|t-mobile\|mint mobile\|cricket\|vi\|bsnl` | `Expenses:Utilities:Phone` |
| `insurance\|geico\|state farm\|allstate\|lic\|hdfc life` | `Expenses:Insurance` |
| `doctor\|hospital\|pharmacy\|cvs\|walgreen\|medical\|dental\|health` | `Expenses:Health:Medical` |
| `rent\|landlord\|property mgmt` | `Expenses:Housing:Rent` |
| `mortgage` | `Expenses:Housing:Mortgage` |
| `atm\|cash withdrawal\|cash wdl` | `Expenses:Fees:ATM` |
| `bank fee\|maintenance fee\|service charge\|annual fee` | `Expenses:Fees:Bank` |
| `salary\|payroll\|direct deposit\|wages` | `Income:Salary` |
| `interest\|int\.pd\|int credit` | `Income:Interest` |
| `dividend` | `Income:Dividends` |
| `refund\|reversal\|cashback\|reward` | `Income:Refund` |
| `transfer` (between own accounts) | Skip or ask - this is an internal transfer |

**Regional narration formats:**

- **India (UPI)**: `UPI/<VPA>/<name>/<ref>` - extract merchant/person name
- **India (NEFT/IMPS)**: `NEFT/<ref>/<name>` or `IMPS/<ref>/<name>`
- **US (ACH)**: `ACH DEBIT <company>` or `ACH CREDIT <company>`
- **US (Check)**: `CHECK #<number>` - ask user who it was to
- **UK (BACS)**: `BACS <reference>` or `FPS <reference>`

For anything that doesn't match a pattern:
1. Tag as `Expenses:Misc` with the full narration as a comment
2. Present it to the user and ask for categorization
3. Remember their choice for future imports from the same payee

### Step 5: Generate ledger entries

Detect the formatting style from the existing journal. If starting fresh,
use this default format:

```ledger
YYYY-MM-DD * Clean Payee Name
  ; Narration: <original bank description>
  ; Imported: YYYY-MM-DD
  Expenses:Category                         <symbol>AMOUNT
  Assets:Bank:<Account>
```

**Formatting rules:**
- **Date format**: match existing journal (default: `YYYY-MM-DD`)
- **Cleared flag**: mark imported transactions with `*` (cleared)
- **Payee**: clean up raw narration into a human-readable name
- **Original narration**: preserve as `; Narration:` comment
- **Import date**: add `; Imported:` metadata for audit trail
- **Amount formatting**: match the commodity format from the journal
- **Elide balancing amount**: let ledger auto-calculate the other side
- **Debits**: target account (expense) gets positive amount, source (bank) is elided
- **Credits**: target account (bank) gets positive amount, source (income) is elided

### Step 6: Present, review, and append

1. **Show** all parsed transactions in a readable table:
   ```
   Date       | Payee              | Category              | Amount
   -----------|--------------------|-----------------------|--------
   2025-03-15 | Netflix            | Expenses:Subscriptions | $15.99
   2025-03-16 | Whole Foods        | Expenses:Food:Groceries| $87.32
   2025-03-17 | ??? UPI/unknown    | Expenses:Misc          | $42.00  <-- NEEDS REVIEW
   ```

2. **Highlight** ambiguous transactions and ask for categorization

3. **Check for duplicates** by comparing date + amount + rough narration match
   against existing entries in the ledger file

4. After user approval, **append** to the ledger file in chronological order

5. **Verify** the journal still parses:
   ```bash
   ledger -f <ledger-file> balance
   ```

6. **Show summary** after import:
   ```bash
   ledger -f <ledger-file> balance --period "this month" Expenses Income
   ```

---

## Reporting

When the user asks about their finances, use these commands. Always substitute
the actual ledger file path.

```bash
# === OVERVIEW ===
ledger -f $FILE balance                                    # Everything
ledger -f $FILE balance Assets Liabilities                 # Net worth
ledger -f $FILE balance Income Expenses                    # Cash flow

# === EXPENSES ===
ledger -f $FILE balance Expenses --depth 2                 # By category
ledger -f $FILE -S "-abs(total)" balance Expenses --depth 3 # Top expenses
ledger -f $FILE -M register Expenses                       # Monthly trend
ledger -f $FILE register Expenses -l "amount >= 1000"      # Large expenses

# === INCOME ===
ledger -f $FILE balance Income                             # Income sources
ledger -f $FILE -M register Income                         # Monthly income

# === TIME-BASED ===
ledger -f $FILE --period "this month" balance Expenses     # Current month
ledger -f $FILE --period "last month" balance Expenses     # Last month
ledger -f $FILE --period "this year" -M balance Expenses   # YTD monthly
ledger -f $FILE -b <date> -e <date> balance                # Custom range

# === SPECIFIC ===
ledger -f $FILE register @"<payee>"                        # By payee
ledger -f $FILE balance Assets:Investments                 # Investments
ledger -f $FILE balance Liabilities                        # Debts
ledger -f $FILE balance Assets:Receivable                  # Money owed to you

# === EXPORT ===
ledger -f $FILE csv                                        # Full CSV export
ledger -f $FILE csv Expenses --period "this month"         # Monthly expense CSV
ledger -f $FILE print --period "this month"                # Clean printable output

# === BUDGET (if periodic transactions defined) ===
ledger -f $FILE --budget balance Expenses                  # Budget vs actual
ledger -f $FILE --unbudgeted balance Expenses              # Unbudgeted spending
```

When presenting reports to the user, add context:
- Compare to previous periods ("Spending is up 15% vs last month")
- Highlight largest categories
- Flag unusual transactions

---

## Manual Transaction Entry

When the user wants to add a transaction manually (not from a PDF):

1. Ask for: date, payee, amount, and category (suggest from existing accounts)
2. Determine source/destination accounts
3. Format and append to journal
4. Verify with `ledger balance`

Example interaction:
```
User: "I spent $45 on groceries at Trader Joe's yesterday"

Generated:
2025-03-28 * Trader Joe's
    Expenses:Food:Groceries                     $45.00
    Assets:Bank:Checking
```

---

## Guidelines

1. **Never modify existing transactions** unless the user explicitly asks
2. **Always verify** after changes: `ledger -f <file> balance`
3. **Duplicate detection**: compare date + amount + narration before appending
4. **Ask before categorizing** ambiguous transactions
5. **Match existing style**: date format, indentation, commodity format, comments
6. **Chronological order**: insert new transactions in date order
7. **Balance verification**: after bulk imports, suggest the user verify their
   bank balance matches: `ledger -f <file> balance Assets:Bank:<name>`
8. **Privacy**: never log or transmit financial data; all processing is local
9. **Learn from corrections**: if the user recategorizes something, apply that
   pattern to future imports from the same payee
