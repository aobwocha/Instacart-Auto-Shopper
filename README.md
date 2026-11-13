# InstaCarter

**InstaCarter turns a weekly grocery list into a ready-to-checkout Instacart shopping list, automatically, on a schedule.** It runs unattended from cron, resolves each item to an Instacart product ID, and posts the full list to the Instacart Connect API through an event-driven pipeline.

```text
$ make run && tail -n 6 logs/run_instacart.log
2026-11-13 07:30:01 - src.handlers.product_resolver - INFO - Resolved product ID 254990 for: Tilda Basmati Rice
2026-11-13 07:30:01 - src.handlers.payload_builder - INFO - Added item to payload: Tilda Basmati Rice (1/12)
...
2026-11-13 07:30:01 - src.handlers.payload_builder - INFO - Payload complete with 12 items, publishing PayloadReadyEvent
2026-11-13 07:30:01 - src.handlers.api_client - INFO - Sending POST request to https://connect.instacart.com/idp/v1/products/products_link
2026-11-13 07:30:02 - src.handlers.api_client - INFO - API request successful: {'products_link_url': 'https://...'}
```

## Motivation

I buy almost the same groceries every week: rice, ground beef, chicken, eggs, bread, a few cans of tomatoes. Rebuilding that cart by hand in the Instacart app every Friday meant searching for the same twelve products and tapping through the same screens. Saved lists and reorder buttons still need me to open the app and remember to do it. So I built **InstaCarter**: a small service that wakes up on a cron schedule, assembles the list, and hands Instacart a finished shopping list, so the weekly shop starts with the cart already filled.

I designed it the way larger production services are built. Each stage is a separate handler on an **in-process event bus**, so resolving products, building the payload, and calling the API stay independent and can be tested one at a time.

## Quick Start

### Prerequisites

- Python 3.9+
- An [Instacart Connect](https://docs.instacart.com/connect/) API key

### 1. Clone and install

```bash
# Clone the repo and create a virtualenv with dependencies
git clone https://github.com/aobwocha/Instacart-Auto-Shopper.git
cd Instacart-Auto-Shopper
make setup
```

### 2. Add your API key

```bash
# config.py loads this with python-dotenv
echo "INSTACART_API_KEY=your_key_here" > .env
```

### 3. Run it

```bash
# The run script writes to logs/, which must exist
mkdir -p logs
make run
```

The result, including the Instacart response, is written to `logs/run_instacart.log`.

## Usage

### How the pipeline works

`src/main.py` creates an event bus, registers three handlers, and publishes one `ItemAddedEvent` per item on the list:

```text
ItemAddedEvent          -> ProductResolver -> ProductIdResolvedEvent
ProductIdResolvedEvent  -> PayloadBuilder  -> PayloadReadyEvent (once every item has resolved)
PayloadReadyEvent       -> APIClient       -> POST to Instacart -> APISuccessEvent / APIRequestFailedEvent
```

- **ProductResolver** maps each item name to its Instacart product ID.
- **PayloadBuilder** collects resolved items and builds the request body only when the whole list is ready.
- **APIClient** posts the payload to Instacart's `products_link` endpoint and logs the response.

### Schedule it with cron

`scripts/run_instacart.sh` finds the project root from its own location, so cron can call it by absolute path. For example, every Friday at 7:30 AM:

```bash
30 7 * * 5 /full/path/to/Instacart-Auto-Shopper/scripts/run_instacart.sh
```

### Change the shopping list

The items and the request template live in `src/config.py`:

- `ID_MAPPING`: item name -> Instacart product ID. Add, remove, or swap entries to change what gets ordered.
- `BASE_PAYLOAD`: list title, store instructions (default: Walmart), link expiry, and pantry-item settings.

### Make commands

- `make setup` - create `venv/` and install dependencies
- `make run` - run the pipeline, logging to `logs/run_instacart.log`
- `make test` - run the test suite, logging to `logs/test_instacart.log`
- `make install-deps` - reinstall dependencies into the existing `venv/`
- `make clean` - remove `venv/` and `__pycache__`

### Known limitations

- Every line item is sent as quantity `1`, unit `kg`; `PayloadBuilder` does not yet use the quantity and unit on `ItemAddedEvent`.
- Success and failure events are only logged, so the process exits `0` either way. Check the log for the outcome.
- An item missing from `ID_MAPPING` publishes a failure event, and the payload is never sent. The current `main.py` reads items from `ID_MAPPING` itself, so this only matters if the list comes from somewhere else.
- `partner_linkback_url` in `BASE_PAYLOAD` is a placeholder (`"string"`). Set it in `src/config.py` if you need a real callback URL.

## Contributing

### 1. Clone the repo

```bash
git clone https://github.com/aobwocha/Instacart-Auto-Shopper.git
cd Instacart-Auto-Shopper
```

### 2. Set up the environment

```bash
make setup
mkdir -p logs
```

### 3. Run the test suite

```bash
# Unit tests for the event bus and handlers, plus an end-to-end integration test
make test
cat logs/test_instacart.log
```

### 4. Submit a pull request

Fork the repo and open a pull request against the `main` branch.
