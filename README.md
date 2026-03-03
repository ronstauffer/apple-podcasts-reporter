# Apple Podcasts Connect Reporter (macOS Setup)

A working setup for Apple's [Reporter](https://podcastsconnect.apple.com/analytics) CLI tool (v2.2) on macOS, including fixes for two issues that prevent it from running on modern Java:

1. **TLS compatibility** — Java 17+ disables cipher suites that Apple's server requires. A custom security config re-enables them.
2. **Patched JAR** — The original `Reporter.jar` has a bug where it assumes Apple's server always returns a `filename` response header when downloading data. Apple's server no longer does this consistently, causing a cryptic "Cannot save file" error. This repo includes a patched JAR with that fixed.

> **Important:** Reporter only works for **paid Apple Podcasts subscriptions**. If your podcasts are free, this tool returns no data — use the web dashboard at [podcastsconnect.apple.com](https://podcastsconnect.apple.com) instead.

---

## Requirements

- macOS (tested on Sequoia 15.x)
- [Homebrew](https://brew.sh)
- Java 17 (see Step 1 — newer versions won't work)
- An Apple ID with two-factor authentication
- An Apple Podcasts Connect account with paid podcast subscriptions

---

## Step 1 — Install Java 17

Java **17 specifically** is required. Newer versions (18+) have TLS issues with Apple's reporting server that can't be easily worked around.

```bash
brew install --cask temurin@17
```

> Do not run `brew install --cask temurin` — that installs the latest version, which won't work.

Verify the install:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java -version
```

You should see `openjdk version "17.x.x"`.

---

## Step 2 — Create Your Properties File

Copy the included template:

```bash
cp Reporter.properties.template Reporter.properties
```

You'll fill in `AccessToken` and `Account` in the steps below. The `Reporter.properties` file is excluded from git via `.gitignore` — never commit it, as it contains your access token.

---

## Step 3 — Get an App-Specific Password

Reporter can't use your regular Apple ID password because of two-factor authentication.

1. Go to [appleid.apple.com](https://appleid.apple.com) and sign in
2. Under **Sign-In and Security**, click **App-Specific Passwords**
3. Click **+**, name it "Reporter", and copy the generated password (format: `xxxx-xxxx-xxxx-xxxx`)

---

## Step 4 — Generate Your Access Token

From the directory containing `Reporter.jar`, run:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java \
  -Djava.security.properties=custom.security \
  -jar Reporter.jar p=Reporter.properties Sales.generateToken
```

When prompted, enter your Apple ID email and your **app-specific password** (not your regular password). Confirm with `y` when asked.

Copy the `AccessToken` value from the output and add it to `Reporter.properties`:

```
AccessToken=your-token-here
```

Tokens expire after 180 days. Regenerate with the same command when needed.

---

## Step 5 — Add Your Account Number

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java \
  -Djava.security.properties=custom.security \
  -jar Reporter.jar p=Reporter.properties Sales.getAccounts
```

This returns your account name and ID number. Add the ID to `Reporter.properties`:

```
Account=your-account-number
```

---

## Running Reports

All commands follow this pattern:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java \
  -Djava.security.properties=custom.security \
  -jar Reporter.jar p=Reporter.properties [COMMAND]
```

Replace `[COMMAND]` with any of the following:

| Command | Description |
|---|---|
| `Sales.getAccounts` | List your accounts |
| `Sales.getVendors` | List vendor numbers (needed for reports) |
| `Sales.getReportVersion Subscriber, Detailed` | Check the latest report version |
| `Sales.getReport [vendor], Subscriber, Detailed, Daily, YYYY-MM-DD` | Download a report |
| `Sales.viewToken` | View your current token and expiry date |
| `Sales.generateToken` | Generate a new access token |

---

## How the Fixes Work

### custom.security

Apple's reporting server requires TLS cipher suites (`TLS_RSA_*`) that Java 17 disables by default. The included `custom.security` file overrides just that restriction.

The Java flag uses a single `=` — this is intentional and important:

```
-Djava.security.properties=custom.security   ✓ overrides only the specified property
-Djava.security.properties==custom.security  ✗ replaces the entire security config, breaking SSL
```

### Patched JAR

The original `Reporter.jar` (v2.2, distributed by Apple at [podcastsconnect.apple.com/analytics](https://podcastsconnect.apple.com/analytics)) calls `HttpURLConnection.getHeaderField("filename")` and passes the result directly to `new FileOutputStream(filename)` with no null check. When Apple's server omits that header, this throws a `FileNotFoundException` that Reporter surfaces as "Cannot save file."

The patched JAR adds a null/empty check and falls back to a safe default filename. No other behavior is changed.

---

## About Reporter.properties

Your `Reporter.properties` file should look like this:

```
AccessToken=your-token-here
Account=your-account-number

Mode=Normal

SalesUrl=https://reportingitc-reporter.apple.com/reportservice/sales/v1
FinanceUrl=https://reportingitc-reporter.apple.com/reportservice/finance/v1
```

This file is in `.gitignore`. Do not commit it or share it — your access token is a secret.
