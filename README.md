# Bangladesh Commodity Price Predictor

A Streamlit web app that **forecasts retail prices** and **estimates prices across districts** for chicken, rice and fish
in Bangladesh. Visitors land on an intro page, then continue with just a name or phone number (no password) to use the
app and earn points. Administrators can upload real monthly prices and retrain the forecast model from inside the app.

**Spatial-aware forecasting experiment.** The Cross-district tab now also has a research experiment that tests
whether neighbouring districts' prices improve forecasting compared with a district's own price history alone (see
"Spatial-aware forecasting experiment" below). It adds to the tab without changing the existing price-ratio estimate,
and the Forecast tab and its model are untouched.

**Version 2.1.** The Submit-a-price tab no longer has a date to pick - a submission is always for the current
month, automatically. Dates shown anywhere in the app (tables, the admin "view all data" and upload preview,
the CSV format guide) now consistently show just year and month, since every price in this app is monthly and
the day was never meaningful.

**Version 2.0.** Added the landing page and name/phone login, points and anomaly-aware submission review (see
"Submission review" and "Points" below), an Admin > Results tab for model accuracy and submission insight, a
"view all data" option in the Forecast and Cross-district admin details, and a visual pass across the whole app.
`data_tools.py` was also commented much more thoroughly throughout, especially around how the model itself works.

## Features

After the landing page and a one-field login (name or phone number - see "Accounts" below), the app has:

| Tab | What it does |
|---|---|
| **Forecast** | Predicts the price of a commodity in a district for 1 to 18 months, starting next month. Shows a chart, a downloadable table, the data basis, and a second chart comparing that commodity's latest price across every district. |
| **Cross-district** | Enter the known price in one district and get an estimate for another district (price-ratio method), with a chart of both districts' real yearly prices. Also contains the **Spatial-aware forecasting experiment**, which compares a forecast model using only a district's own history against the same model with neighbouring districts' prices added. |
| **Budget planner** | Enter a budget and a time horizon; ranks district/commodity combinations by predicted price rise, keeping only what the budget can afford. Fish is excluded by default (it doesn't store like rice or other durable goods). Costs one free run or one point per use - see "Points" below. |
| **Submit a price** | A (district, commodity, price) submission for the current month - the month is always "now", it isn't a choice. One that matches the recent trend for that item is added immediately and earns a point right away; one that looks like an unusual spike or drop is held for an administrator to check first - see "Submission review" below. |
| **Admin** (password) | Live dataset statistics by commodity and category, a Results tab with current model accuracy and submission/points insight, upload monthly prices as CSV with automatic checks, review flagged public submissions, undo an upload, retrain the forecast model with a safe test before replacing it, and see the numbers behind each result (including a "view all data" option in the Forecast and Cross-district tabs). |

## Quick start

Python 3.12 or newer is recommended.

```bash
python -m venv venv
venv\Scripts\activate            # Windows      (macOS / Linux: source venv/bin/activate)
pip install -r requirements.txt
streamlit run app.py
```

Run the last command from the project folder (the folder that contains `app.py`).

### Admin password (one-time setup)

The Admin tab stays locked until you set a password. Copy `.streamlit/secrets.toml.example` to
`.streamlit/secrets.toml`, choose your own password, and restart the app:

```toml
ADMIN_PASSWORD = "your-password"
```

`secrets.toml` is listed in `.gitignore`, so it is never committed. You can also set an `ADMIN_PASSWORD`
environment variable instead.

## How it works

**Data.** The raw files (`data/raw`) come from the Department of Agricultural Marketing (DAM) and contain
*one retail price per market per year* (2025 and 2026). They are cleaned and averaged per district
(`data/processed/district_prices_clean.csv`).

**Forecast.** The DAM files have no monthly prices, so a monthly series is *generated* from the two yearly averages
(a straight line plus small random noise, `data/processed/monthly_hybrid.csv`). An XGBoost model learns to predict
next month's price from this month's price, the three months before, the month number and the district/commodity.
Longer forecasts feed the model's own predictions back in, so they become less reliable the further ahead they go.

**Real data replaces generated data.** When an administrator uploads at least **6 consecutive months** of real prices
for a district and commodity, that series uses only the real prices from then on. Everything else keeps using the
generated series. The app always shows which one a forecast is based on.

**Cross-district.** For every year in which both districts have a price, the target price is divided by the source price.
The average of these ratios is multiplied by the price you enter. It uses only the real yearly prices.

**Spatial-aware forecasting experiment (Cross-district tab).** A separate experiment for the research question
*"does adding neighbouring district prices improve forecasting compared with using only the target district's own
history?"* It is not the price-ratio estimate above: here neighbouring prices are given to the model as input
features, and the model has to learn whether they help. The same XGBoost model (same settings, same seed) is
trained twice on the same rows, with the same train/test split and the same target:

| Feature set | Inputs |
|---|---|
| **Set A - baseline** | Exactly the Forecast model's inputs: this month's price, lag1-lag3, month number, time index, district and commodity. |
| **Set B - spatial-aware** | Everything in Set A, plus: `nbr_mean_price` (average price across the district's geographic neighbours, this month), `nbr_mean_lag1` (the same average one month earlier), `div_mean_price` (average across the rest of the district's division) and `ref_gap` (this district's price minus Dhaka's). |

So any difference in accuracy can only come from the spatial columns. Every spatial value is taken from the month
the forecast is made or the month before, never from the month being predicted.

- **Neighbours** come from a fixed geographic list, `data/reference/district_neighbors.csv` (one
  `target_district, neighbour_district` pair per line, both directions listed). They are *not* chosen from price
  correlations. Edit the file to add or remove a neighbour; nothing else needs to change.
- **Divisions** are read from the `Division` column of the DAM raw files, so there is no second hand-typed list.
- **Neighbour averages, not one column per neighbour.** The model is shared across every district, so a column like
  "Gazipur price" would only mean something for Dhaka's rows. An average over "my neighbours" means the same thing
  on every row. The individual neighbour prices are still shown in the app.
- **Missing neighbours** (a neighbour that does not sell the commodity) are left out of the average; if none sells
  it, the value is left empty, which XGBoost handles itself. This keeps exactly the same rows in Set A and Set B.
- **Horizon** is 1-6 months. Horizon 1 is the Forecast model's own target (next month). Longer horizons are
  predicted directly (month t to month t+h), because a step-by-step spatial forecast would also need every
  neighbour's future prices.
- **Testing.** The latest 2-6 months (default 3) are held out. Both models are trained only on prices that would
  have been known when the first test forecast was made, then scored with MAE, RMSE, MAPE and R² (R² only with
  5 or more test rows). "Improvement" is (A - B) / A, so a positive number means Set B made smaller errors.
  Results can be shown for one district and commodity, one commodity across all districts, or everything; the
  single-series view has only as many test rows as test months, so conclusions should rest on the wider views.
- The app also shows both models' next prediction, a chart of real vs predicted prices over the test months, and
  how much of Set B's learning came from the spatial features (feature importance).

The experiment only starts after pressing **Run experiment** (training takes a few seconds, and Streamlit runs
every tab on each click); after that, results are cached and changing the district or commodity is instant.

**Reading the result on generated data.** On the current dataset Set A and Set B score within about 1% of each
other and the spatial features carry almost no importance. That is expected: each generated monthly series has
its own independent random noise, so neighbouring districts share a trend but no month-to-month movement. This is
not evidence for or against spatial effects in real markets. The experiment uses the same monthly data as the rest
of the app, so once real monthly prices are uploaded it runs on them automatically.

**Category.** Every commodity is tagged Chicken, Rice or Fish - directly from which raw file it came from
(`chicken.csv`, `rice.csv`, `fish.csv`), never guessed from the name. This powers the Admin Statistics tab and
the Budget planner's fish exclusion.

**Budget planner.** Runs the same forecast used by the Forecast tab across every eligible district/commodity
combination, ranks them by predicted % price change over the chosen horizon, and drops anything the budget can't
afford at least 1 kg of today. Like the Forecast tab, its ranking is only as reliable as the data behind it: on
series still using the generated monthly data, an apparent "fast riser" often just reflects the 2025-2026 yearly
average gap rather than an observed trend. It has no concept of storage cost or shelf life beyond the fish
exclusion, so treat it as a starting point for comparison, not financial advice.

**Submission review.** A public submission is always for the current month - there is no date to pick, since it
represents the price right now - and is checked first with the same rules as an admin CSV upload
(`validate_upload` - unknown names, future dates, and a price more than `MAX_PRICE_FACTOR` (3x) away from that
commodity's overall median are rejected outright). What happens next is a finer check, `assess_submission`: the
price is compared against *that specific district and commodity's own* last known price, not the overall median -
and a rise and a fall are judged differently, because in a real market a price rising by some amount is far more
often ordinary than falling by that same amount is:

| Move (vs. that series' own last price) | Result |
|---|---|
| Rise up to 30%, or fall up to 15% | Added immediately, point credited right away |
| Rise 30-50%, or fall 15-30% | Held for review, labelled "moderate" |
| Rise 50%+, or fall 30%+ | Held for review, labelled "severe" |
| No earlier price for this series at all | Always held for review (nothing to compare it to yet) |

Both "moderate" and "severe" are held the same way and need the same administrator decision - "severe" is just a
stronger label in the Admin > Community submissions list, so a bigger move stands out at a glance. Anything held
is shown to an administrator (Admin > Community submissions, whose tab label shows a live count) with the exact
% move, before it becomes real data or earns its point. `RISE_ACCEPT_LIMIT`, `FALL_ACCEPT_LIMIT`,
`RISE_SEVERE_LIMIT` and `FALL_SEVERE_LIMIT` in `data_tools.py` control all four numbers above. This mirrors the
trust level of the rest of the app: admin CSV uploads skip this check entirely, since admins are already trusted.

**Accounts.** There are no passwords for regular use - just a name or phone number, entered once after the landing
page. It works like a lightweight, unique username: an identifier that has been used before logs back into the
same account (with its points intact); one that hasn't creates a new account on the spot. This is a light throttle
tied to what someone types, not a verified identity - nothing stops someone from typing a different name to reset
their free trials, or someone else's name to use their points. A real password-based account system would close
that gap, but it's a much bigger addition - registration, credential storage, session handling - than this
project's single shared admin password otherwise needs. If abuse becomes a real problem, that trade-off is worth
revisiting. Logging in (or registering) is `data_tools.login_or_register`, tracked in `data/processed/users.csv`.

**Points.** Running the Budget planner costs one credit: the first 3 runs per account are free
(`FREE_BUDGET_PLANS` in `data_tools.py`), then it costs 1 point, tracked in `data/processed/points_ledger.csv`.
A point is earned the moment a submission is added - whether that's immediately (it matched the recent trend) or
later, when an administrator approves a flagged one.

**Admin > Results.** Shows the accuracy of the model that is *currently live* (`evaluate_current_model` - it tests
the running model against the latest months it was not trained to predict, without training anything new), a
chart of accuracy across successive retrains, a breakdown of how public submissions have been handled, and a
leaderboard of who has earned the most points. This is a reporting view only; it changes nothing about the model
or anyone's points - it exists to make the app's own behaviour easy to check and explain.

## Admin tab: turning it on

The Admin tab's password is separate from the name/phone login everyone else uses to reach the app - it locks
until a password exists, exactly as before this became a login-gated app. Set one **once**:

1. In the project folder, open `.streamlit/secrets.toml.example`.
2. Save a copy in the same folder named exactly `secrets.toml` (not `secrets.toml.txt`).
3. Put one line inside it, with your own password in the quotes:
   ```toml
   ADMIN_PASSWORD = "your-password"
   ```
4. Restart the app (`Ctrl+C`, then `streamlit run app.py`) and open the Admin tab.

Instead of the file, you can set an environment variable named `ADMIN_PASSWORD` before starting the app.
`secrets.toml` is listed in `.gitignore`, so it is never shared.

While logged in, "Details (administrators only)" sections appear in the Forecast and Cross-district tabs showing the
numbers behind each result, plus a "View all data" checkbox that shows the complete underlying dataset (every
district and commodity, not just the current selection). They disappear again when you press **Lock**.

## Admin tab: uploading data

Upload a CSV with the columns `date, location, commodity, price_per_kg` (see the *File format* sub-tab in the app).
Every price in this app is a **monthly** price, so `date` only needs a year and month (`2026-08`); a full date
(`2026-08-01`) is also accepted, but the day is always ignored - `2026-08-15` and `2026-08-01` both become the
same month, `2026-08`. Every row is checked before anything is saved: date format, future months, prices, and
district and commodity names (wrong spellings are rejected with a suggestion). Saving makes a backup first, and
**Undo last upload** restores it. **Retrain model** trains a new model, tests it on the latest 3 months against
the baseline "next month = this month", and only replaces the current model when you confirm. The old model is
kept in `models/backups/`. The **Results** tab is where to check on accuracy afterwards - both the live model's
current accuracy and the history of every retrain - without needing to retrain again just to see it.

## Project structure

```
app.py                      the user interface (Streamlit)
data_tools.py               all calculations: data loading, upload checks, forecasting, training
requirements.txt            packages needed to run the app
requirements-dev.txt        extra packages for the notebooks and the tests
.streamlit/config.toml      theme and server settings
data/raw/                   DAM price files (chicken, rice, fish)
data/processed/             cleaned district prices, the generated monthly series, users, points and pending submissions
data/reference/             district_neighbors.csv - the editable geographic neighbour list for the spatial experiment
models/                     the trained forecast model (JSON) and its list of input columns
notebooks/                  01_data_preparation.ipynb, 02_train_forecast_model.ipynb
tests/                      automated tests
```

`data_tools.py` is commented heavily on purpose, especially the "Training the forecasting model" and
"Forecasting" sections - it explains what each input feature means, why XGBoost, what its settings do, and
exactly how a multi-month forecast is built one predicted month at a time. It is written to be read top to
bottom by someone who has not seen the project before, including for explaining the model to someone else.

## Rebuilding the data and the model

```bash
pip install -r requirements-dev.txt
jupyter notebook
```

Run `notebooks/01_data_preparation.ipynb` first, then `notebooks/02_train_forecast_model.ipynb`.

## Tests

```bash
python -m pytest -q
```

The tests cover the upload checks, the real-versus-generated rule, undo, the forecast (including that it always starts
next month and uses exactly the inputs the model was trained on), the cross-district estimate, the data files, and the spatial experiment (the neighbour list is symmetric and complete,
Set B is Set A plus only the spatial features, and no training price comes from after the test period).

## Troubleshooting

**"The model files could not be read" or "is missing"** - rebuild them:
`pip install -r requirements-dev.txt`, open `notebooks/02_train_forecast_model.ipynb` and run *Kernel > Restart & Run All*.
This takes a few seconds and needs no download.

**The Admin tab says access is not set up** - create `.streamlit/secrets.toml` as described above and restart the app.

## Known limitations

- The monthly history is generated from yearly averages, so it contains no real month-to-month movement or seasonality.
  Forecasts are therefore only indicative until real monthly prices are uploaded.
- The Cross-district tab assumes the price gap between two districts stays the same; the ratio changed between 2025 and 2026.
- The spatial-aware experiment cannot show a real spatial effect on the generated monthly data (see its section above).
  A few river borders in `district_neighbors.csv` were judgment calls and are worth checking against a map.
- Tree-based models such as XGBoost cannot extrapolate beyond the price range they have seen.
- Predictions can be wrong and must not be the only basis for a decision.
- The name/phone login has no password, so it is a light throttle on repeat use, not a verified identity (see "Accounts" above).
- Staying logged in relies on Streamlit's session state, which resets on a full page reload - logging back in with the
  same name or phone number restores the same account and points, but there is no "remember me" across browser restarts.
- The anomaly check in "Submission review" compares against one prior price point, so a series with genuinely volatile
  real prices could see legitimate updates flagged more often than necessary; `RISE_ACCEPT_LIMIT`, `FALL_ACCEPT_LIMIT`,
  `RISE_SEVERE_LIMIT` and `FALL_SEVERE_LIMIT` can all be tuned for that.
