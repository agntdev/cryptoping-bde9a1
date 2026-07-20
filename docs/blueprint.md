# CryptoPing — Bot specification

**Archetype:** custom

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

A Telegram bot for tracking cryptocurrency prices with customizable alerts. Users manage private watchlists, set price-threshold and percent-change alerts, receive push notifications (with spam suppression), and get optional morning digests. The bot owner gains access to aggregate usage metrics like active users and most-triggered alerts.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Individual crypto traders
- Crypto market watchers
- Telegram bot owners seeking analytics

## Success criteria

- Users can add/remove coins to watchlists via buttons and typed tickers
- Price and percent-change alerts trigger with configured rules
- Admin /stats command shows active users and top alerts
- Morning digest delivered at user-specified time
- Quiet hours suppress alerts but preserve digest delivery

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Initialize user profile, detect timezone, and show main menu
- **/price** (command, actor: user, command: /price) — Show current price of specified coin or full watchlist summary
- **/stats** (command, actor: admin, command: /stats) — Display aggregate metrics to bot owner
- **Add Bitcoin** (button, actor: user, callback: watchlist:add:BTC) — Quick-add preset coin to watchlist
- **Add custom ticker** (button, actor: user, callback: watchlist:custom) — Prompt user to type a custom ticker symbol

## Flows

### Onboarding
_Trigger:_ /start

1. Detect user timezone
2. Prompt for morning digest time
3. Configure quiet hours
4. Show initial watchlist

_Data touched:_ user_profile

### Price Alert Creation
_Trigger:_ Add price alert button

1. Select coin from watchlist
2. Choose alert direction (above/below)
3. Enter target price
4. Confirm and save rule

_Data touched:_ alert_rule

### Morning Digest
_Trigger:_ Scheduled time

1. Check user's local time
2. Fetch 24h price changes
3. Format digest with notable movements
4. Send to user if outside quiet hours

_Data touched:_ user_profile, notification_event

### Alert Suppression
_Trigger:_ Price threshold crossed

1. Check if within quiet hours
2. Verify price source reliability
3. Apply cooldown period for rule
4. Send alert if conditions met

_Data touched:_ alert_rule, notification_event

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **user_profile** _(retention: persistent)_ — User-specific settings and preferences
  - fields: telegram_id, timezone, quiet_hours, digest_time, fiat_currency
- **watchlist_item** _(retention: persistent)_ — Cryptocurrency ticker with display name and rules
  - fields: ticker, display_name, alert_rules
- **alert_rule** _(retention: persistent)_ — Price threshold or percent change alert configuration
  - fields: rule_type, target_value, direction, window_minutes, enabled
- **notification_event** _(retention: persistent)_ — Record of triggered alerts for metrics
  - fields: timestamp, user_id, ticker, old_price, new_price, percent_change, rule_id
- **admin_metrics** _(retention: persistent)_ — Aggregate usage statistics
  - fields: active_users_30d, top_alerts, total_alert_triggers

## Integrations

- **Telegram** (required) — Bot API messaging and inline buttons
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- /stats command to view metrics
- Default settings for new users (timezone, digest time, quiet hours)

## Notifications

- Push alerts for price thresholds/percent changes
- Morning digest summary
- Error notifications for price source failures

## Permissions & privacy

- User data is private and only accessible to the user and owner
- Admin metrics do not expose individual watchlists
- Data retention: 90 days for alert logs, persistent for user settings

## Edge cases

- Price source unavailable (retry with backoff)
- Misspelled tickers (suggest corrections)
- Alert flapping during cooldown period
- Morning digest scheduled during quiet hours (queue for delivery after)

## Required tests

- Verify alert suppression during quiet hours
- Test digest delivery after quiet hours end
- Validate cooldown prevents repeated alerts
- Confirm admin metrics show correct top alerts

## Assumptions

- Default fiat is USD
- Percent-change window defaults to 1 hour
- Morning digest is queued if scheduled during quiet hours
- Admin metrics exclude private user data
