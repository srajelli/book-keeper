# book-keeper

A Claude Code skill for double-entry bookkeeping with [ledger-cli](https://ledger-cli.org/).

Import bank statement PDFs, auto-categorize transactions, and maintain a plain-text accounting journal with full visibility over your income and expenses.

## Features

- **PDF statement import** — parse bank statements using [liteparse](https://github.com/run-llama/liteparse), including password-protected PDFs via qpdf
- **Auto-categorization** — recognizes common merchants and narration patterns (UPI, NEFT, IMPS, ACH, BACS, etc.)
- **Any currency** — works with $, EUR, GBP, INR, JPY, or any ledger-cli commodity
- **Any bank** — supports Indian, US, UK/EU, and international statement formats
- **Personal or business** — adapts account hierarchy to your use case
- **Rich reporting** — balance sheets, expense breakdowns, monthly trends, cash flow, budget tracking
- **First-run setup** — bootstraps a new ledger file if you don't have one
- **Privacy-first** — all processing is local, no data leaves your machine

## Prerequisites

- [ledger-cli](https://ledger-cli.org/) — `brew install ledger` (macOS) or `apt install ledger` (Linux)
- [liteparse](https://github.com/run-llama/liteparse) — `npm i -g @llamaindex/liteparse` (for PDF parsing)
- [qpdf](https://qpdf.sourceforge.io/) — `brew install qpdf` (optional, for encrypted PDFs)

## Installation

### Via skills CLI (recommended)

```bash
npx skills add srajelli/book-keeper
```

### Via Claude Code plugin marketplace

```
/plugin marketplace add srajelli/book-keeper
```

### Manual

Clone and copy the skill into your Claude Code config:

```bash
git clone https://github.com/srajelli/book-keeper.git
cp -r book-keeper/skills/book-keeper ~/.claude/skills/
```

The skill auto-installs missing prerequisites (ledger, liteparse, qpdf) on first run — just confirm when prompted.

## Usage

Invoke the skill with `/book-keeper` or just ask about your finances:

```
/book-keeper
> Import my bank statement from ~/Downloads/statement.pdf

> Show me my expenses this month

> I spent $45 on groceries at Trader Joe's yesterday

> What's my net worth?

> Show travel expenses for the last 3 months
```

## How it works

1. **Parse** — extracts text from bank statement PDFs using liteparse
2. **Identify** — detects bank format, date format, columns, and currency
3. **Categorize** — matches transaction narrations to expense/income categories
4. **Review** — presents transactions for your approval, highlights ambiguous ones
5. **Append** — adds approved transactions to your ledger file in chronological order
6. **Verify** — runs `ledger balance` to confirm everything balances to zero
7. **Report** — shows a summary of what was imported

## Account structure

The skill adapts to your existing ledger accounts. If starting fresh, it suggests a standard hierarchy:

```
Assets:Bank:<Institution>
Assets:Cash
Assets:Investments:<Type>
Expenses:Food:Groceries
Expenses:Food:Dining
Expenses:Transport:*
Expenses:Travel:*
Expenses:Shopping:*
Expenses:Utilities:*
Expenses:Entertainment:*
Income:Salary
Income:Interest
Liabilities:Credit Card:<Name>
Liabilities:Loan:<Name>
```

## License

MIT
