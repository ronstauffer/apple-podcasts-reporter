# Apple Podcasts Connect Reporter Setup

A command-line tool for downloading Apple Podcasts **paid subscription** analytics. If your podcasts are free (no Apple Podcasts subscription tier), this tool won't return any data — use the web dashboard at [podcastsconnect.apple.com](https://podcastsconnect.apple.com) instead.

## What You Need

- macOS
- [Homebrew](https://brew.sh)
- An Apple Podcasts Connect account with paid podcast subscriptions
- Two-factor authentication enabled on your Apple ID (you'll need an app-specific password)

## Step 1 — Install Java 17

Install Java **17 specifically**. Newer versions (18+) have TLS compatibility issues with Apple's reporting server.

```bash
brew install --cask temurin@17
```

Do not use `brew install --cask temurin` — that installs the latest version, which won't work.

## Step 2 — Get an App-Specific Password

Because Apple IDs use two-factor authentication, Reporter can't use your regular password.

1. Go to [appleid.apple.com](https://appleid.apple.com) and sign in
2. Under **Sign-In and Security**, click **App-Specific Passwords**
3. Click **+**, name it "Reporter", and copy the generated password (`xxxx-xxxx-xxxx-xxxx`)

## Step 3 — Generate Your Access Token

From this directory, run:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java -Djava.security.properties=custom.security -jar Reporter.jar p=Reporter.properties Sales.generateToken
```

Enter your Apple ID email and your **app-specific password** (not your regular password) when prompted. Confirm with `y` when asked.

Your token will be printed to the screen. Copy the `AccessToken` value and add it to `Reporter.properties`:

```
AccessToken=your-token-here
```

Tokens expire after 180 days. Regenerate using the same command.

## Step 4 — Add Your Account Number

Run:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java -Djava.security.properties=custom.security -jar Reporter.jar p=Reporter.properties Sales.getAccounts
```

This returns your account name and number. Add it to `Reporter.properties`:

```
Account=your-account-number
```

## Running Reports

All commands follow this pattern:

```bash
/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home/bin/java -Djava.security.properties=custom.security -jar Reporter.jar p=Reporter.properties Sales.COMMAND
```

Useful commands:

```bash
Sales.getAccounts                          # your account info
Sales.getVendors                           # your vendor numbers (needed for reports)
Sales.getReportVersion Subscriber, Detailed  # latest report version
Sales.getReport [vendor], Subscriber, Detailed, Daily, YYYY-MM-DD  # download a report
Sales.viewToken                            # check your token and expiry date
Sales.generateToken                        # generate a new token
```

## Why custom.security?

Apple's reporting server uses TLS cipher suites that Java 17 disables by default (`TLS_RSA_*`). The included `custom.security` file re-enables them.

**Important:** the Java flag must use a single `=`, not `==`:
- `=` overrides only the specific property, keeping all SSL providers intact
- `==` replaces the entire security config, breaking SSL completely

## Why is the JAR patched?

The original `Reporter.jar` (v2.2, built in 2017) has a bug where it assumes Apple's server response always includes a `filename` header when downloading data. Apple's server no longer consistently includes this header, causing a `FileNotFoundException` shown as "Cannot save file." The JAR included here has that method patched to handle missing or null filenames gracefully.

## Reporter.properties

The `Reporter.properties` file contains your access token — **never commit it to a public repository or share it.** It's excluded via `.gitignore`.

The file should look like this (fill in your values):

```
AccessToken=
Account=

Mode=Normal

SalesUrl=https://reportingitc-reporter.apple.com/reportservice/sales/v1
FinanceUrl=https://reportingitc-reporter.apple.com/reportservice/finance/v1
```
