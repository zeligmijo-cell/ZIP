# MNQ Desk — personal Windows trading app

## Start here

1. Extract the entire ZIP into a folder on your Windows PC.
2. Open **MNQ Desk.exe**. Python does not need to be installed.
3. On **Connections**, enter your TopstepX username and API key, plus your Anthropic API key. All API addresses, model and optional workspace ID are editable here.
4. Click **Connect & find accounts**, then choose your intended account and active MNQ contract. Use a Topstep Practice account for initial broker testing. The app supports eligible Practice, Trading Combine and Express Funded simulated accounts. Topstep Live Funded accounts are blocked.
5. Write a strategy paragraph in **Strategy**. It is intentionally blank initially. Explain exact entry conditions and when to stay out. This version supplies completed OHLCV candles and the latest polled price, not order-book data, news, or an economic calendar. Strategies requiring unavailable inputs should return HOLD.
6. Review every field in **Risk & schedule**, particularly quantity, risk cap, session loss guard and account balance floor. The example defaults do not encode your Topstep account rules. An Express Funded account may start near $0: a negative balance floor can be appropriate, but must be based on its actual current maximum-loss threshold. Update the floor as that threshold changes.
7. Use **Preview only** first. Press **Start** to see Claude decisions without broker orders. Preview uses paid Anthropic requests; it is not a simulated-fill backtest.
8. For automatic broker orders, enable **Auto OCO Brackets** in TopstepX → Settings → Risk Settings while flat. Check its confirmation in the app, choose **Broker orders**, then press **Start** and confirm the displayed account, contract and size. The bot then places qualifying entries without asking about each one.

The app opens stopped in Preview mode every time. Editing settings requires stopping first. Your API keys are masked and saved using Windows DPAPI encryption for your current Windows user. Other settings, strategy, order metadata and activity logs are stored in `%LOCALAPPDATA%\MNQDesk`. Do not delete the execution ledger to bypass an unresolved order. Protect your Windows account and device; encryption does not protect against programs running as that same user.

## Connections and costs

**TopstepX / ProjectX is both the data provider and execution connection.** Tradovate is not required and is not implemented in this download. Editing the API address changes the TopstepX-compatible endpoint; it cannot turn the app into a Tradovate client. The connection class can be extended in source later for a different broker.

Topstep's currently listed API cost is $29/month, or $14.50/month with recurring discount code `topstep`. This is separate from Topstep account fees. Subscribe through ProjectX, link the subscription in TopstepX, and generate the key in TopstepX. Account eligibility and API fees can change; check the linked guide before purchase.

Claude API usage is separately billed by Anthropic. A Claude chat subscription does not replace an API key. The model field starts with `claude-haiku-5-5`; replace it with a supported model ID available to your Anthropic account if necessary. The session call cap limits the number of requests, not their dollar cost. Set an additional budget in Anthropic Console. The app shows input/output token counts in Activity.

Source: https://help.topstep.com/en/articles/11187768-topstepx-api-access

## What the bot does

- Polls TopstepX one-second completed bars on a five-second cycle while running inside its trading window; network requests and Claude inference add time to each cycle. The last close is the displayed price observation. This is periodic live-data polling, not a tick-stream connection. A fresh observation is required again after inference before any entry is submitted.
- Retrieves the chosen completed minute candles for Claude. Evaluates at most once per new candle, subject to the session call cap.
- Sends the strategy, candles, price and fixed trade settings to Anthropic. No Topstep key, account ID, balance, or order access is given to Claude. Claude can propose BUY, SELL or HOLD only.
- Uses fixed contract quantity, stop distance and target distance from the app's settings. The paragraph cannot override these. This version supports market entries and fixed broker brackets only; it does not implement arbitrary exits, trailing stops, scaling or reversals described in a paragraph.
- Rechecks account exposure, eligibility, price freshness, price movement, active contract and remaining risk room after inference. A decision may be discarded if conditions changed while Claude was responding.
- Requires the whole account to be flat with no active/pending/suspended orders before a new entry. Use a dedicated account and avoid manual trading or another bot while it is running.
- Sends a market entry with a native stop-loss Stop bracket and a take-profit Limit bracket. A returned order ID alone is not considered proof of success or a fill. The app checks the success response and retains the entry in its ledger for broker reconciliation.
- For a bot entry that has filled, checks for a working stop covering the position after a ten-second grace period. If it cannot verify protection, it attempts to flatten MNQ and stops. If any action fails, check TopstepX directly.
- Saves an order intent before transmission. Order calls are never retried automatically. A network failure or unexpected response stops the worker. If the broker accepted the order, restarting cannot blindly resubmit it.
- Tracks session entry reservations, processed candles and Claude-call budgets across restarts. A reservation may count even if a submission was rejected or never reached the broker; this is deliberate.
- Queries account realized P&L including reported fees/commissions approximately every 30 seconds, and adds selected-contract MNQ unrealized P&L estimated from the polled price. This is an entry guard, not an exact broker risk calculation. Other instruments' unrealized P&L is not included.
- Uses Chicago time, including daylight saving. Session budgets reset at 17:00 Chicago. The configurable entry window is weekdays only, same-day, and must end no later than 15:00. It does not automatically track early closes, holidays or prohibited news windows.

## Stop, close and unresolved orders

**Stop** stops future decisions/entries; a request already transmitted can still complete. It leaves existing positions and broker brackets in place. A daily loss guard or connection error also stops new entries without automatically flattening the position. Scheduled flattening will no longer run after the worker stops.

**Stop + flatten MNQ** stops the worker and attempts to close the selected contract on the selected account, then cancels remaining orders on that contract only after flat is confirmed. This includes any manual position on that same contract. It does not close other contracts. Pending market entries require resolution through TopstepX first. Always read the Activity result and verify broker state.

At the configured window end, a running broker-mode worker attempts the same flatten operation and stops. Your PC must be awake, online, and the app must still be running. Broker-held brackets can remain active after the app closes, but the app's scheduled flatten and local guards cannot. Keep TopstepX available for direct monitoring and emergency action.

If an order remains unresolved, use **Activity → Reconcile previous order** with the same API address, account and contract. Reconciliation clears the lock only when a matching unique order tag is visible, its status is terminal, MNQ is flat, and there are no active orders for that contract. A missing tag is not treated as proof that an order failed. If no matching order appears, the lock stays in place: inspect the broker order log and resolve with broker support before changing the ledger. No automatic restart or recovery is attempted.

## Limits and validation

This is a first implementation for personal use, not a proven trading strategy. Claude's interpretation of a paragraph is probabilistic. Deterministic rules should be added and tested once the strategy is defined. No claim of profitability is made. Stops can slip and account limits can be breached despite local risk settings.

The app does not implement Topstep's trailing drawdown, scaling plan, payout rules, consistency rules, news restrictions or a holiday calendar. Configure Topstep's platform-side limits as well. Topstep requires order activity on your personal device and prohibits VPS/VPN/remote-server order flow. This app does not upload a hosted trading worker.

The included automated checks use mocked APIs. No real API keys, paid Claude calls or broker orders were used during build validation. End-to-end behavior, fills, native brackets, rate limits, authentication and your account entitlements must be verified in a Practice account before using a Combine or Express Funded account.

The executable is unsigned; Windows may show its normal unknown-publisher prompt. The complete source is included under `source/` for inspection and rebuilding.

## Source and rebuild

Python 3.12+ with Tkinter, `tzdata`, and PyInstaller are needed only for source development:

```
python -m pip install -r requirements-build.txt
python -m unittest test_core -v
python app.py
python -m PyInstaller --noconfirm --clean --onefile --windowed --name "MNQ Desk" --collect-all tzdata app.py
```

ProjectX endpoint reference: https://api.topstepx.com/swagger/index.html

ProjectX native brackets: https://gateway.docs.projectx.com/docs/api-reference/order/order-place/

Anthropic API: https://platform.claude.com/docs/en/api/overview

MNQ contract specifications: https://www.cmegroup.com/markets/equities/nasdaq/micro-e-mini-nasdaq-100.contractSpecs.html
