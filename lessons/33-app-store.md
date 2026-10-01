---
id: app-store
title: Shipping to TestFlight & the App Store
part: 7
minutes: 35
summary: Take your app from Xcode to users' iPhones. Covers the Developer Program, signing, versioning, archiving and uploading, TestFlight, App Store Connect, privacy requirements and App Review.
---

:::goals
- A pre-release checklist
- The Apple Developer Program, bundle IDs, certificates and automatic signing
- Version and build numbers
- Archiving and uploading with Xcode
- TestFlight beta testing
- The App Store listing: screenshots, privacy details, age rating
- App Review and releasing
:::

## Pre-release checklist

- [ ] **Bundle identifier** is final (e.g. `com.yourname.recipebox`). It can't change after release.
- [ ] **App icon** (1024×1024, with dark and tinted variants) and **display name** are polished.
- [ ] Works on the **smallest and largest** iPhones, in **landscape** if supported, and on **iPad** if you ship there.
- [ ] Handles **no internet**, **dark mode**, the **largest Dynamic Type sizes**, **VoiceOver** and being terminated in the background.
- [ ] Every permission has a clear **usage description**.
- [ ] No debug-only code: test buttons, verbose logging, hard-coded test accounts.
- [ ] A **privacy policy** URL (required for every app).
- [ ] Tested a **Release** build on a real device (e.g. via TestFlight).

## The Apple Developer Program

To distribute on the App Store or TestFlight you need a paid **[Apple Developer Program](https://developer.apple.com/programs/)** membership (renewed yearly). Individuals enrol with their Apple Account; organisations also need a D-U-N-S number. Enrolment can take a day or two.

## Signing, simply

iOS only runs apps **signed** by a certificate Apple issued. Signing involves:

- a **certificate** (proves who you are),
- an **App ID** (your bundle identifier, registered with Apple),
- a **provisioning profile** (ties the certificate, App ID, capabilities and devices together).

With **Automatically manage signing** ticked in *Signing & Capabilities* and your paid team selected, **Xcode creates and renews all of this for you**. You'll rarely need to touch the developer portal manually.

## Version and build numbers

In the target's **General** tab:

- **Version** (`CFBundleShortVersionString`), e.g. `1.0.0`: what users see on the App Store.
- **Build** (`CFBundleVersion`), e.g. `1`, `2`, `3`: must be **unique and increasing** for each upload of that version.

Typical flow: version `1.0.0` builds 1, 2, 3 go to TestFlight; build 3 is released. Then version `1.0.1` starts again at build 1 (or keep incrementing, either works).

## Create the app in App Store Connect

1. Go to **[App Store Connect](https://appstoreconnect.apple.com)** → **Apps** → **+** → **New App**.
2. Choose iOS, enter the **name** (must be unique on the App Store), primary language, your **bundle ID**, and an SKU (any internal reference, e.g. `recipebox-001`).

## Archive and upload

1. In Xcode, set the run destination to **Any iOS Device (arm64)**.
2. **Product → Archive**. Xcode builds an optimised release version.
3. The **Organizer** window opens. Select the archive and click **Distribute App** → **App Store Connect** → **Upload**.
4. Xcode validates, signs and uploads. After a few minutes of processing, the build appears in App Store Connect under **TestFlight**.

:::note Xcode Cloud
Xcode Cloud (in the **Integrate** menu / Report navigator) can build, test and upload automatically whenever you push to Git. It's worth setting up once you're shipping regularly.
:::

## TestFlight

**TestFlight** lets people install beta builds through the TestFlight app:

| Group | Who | Review needed |
|---|---|---|
| **Internal testers** | Up to 100 members of your App Store Connect team | No: available minutes after processing |
| **External testers** | Up to 10,000 people, invited by email or a **public link** | A light Beta App Review for the first build of each version |

Add a "What to Test" note for each build. Testers can send feedback and screenshots straight from the app, and crashes are reported in Xcode's **Organizer → Crashes**.

## The App Store listing

In App Store Connect, fill in the version page:

- **Screenshots**: required for the largest iPhone size (and iPad if supported). Take them in the Simulator ([[Cmd]] + [[S]]) or design framed ones.
- **Promotional text, description, keywords, support URL, marketing URL.**
- **App Privacy** ("nutrition labels"): declare what data your app and any SDKs collect, and how it's used. It must be accurate.
- **Age rating** questionnaire, **category**, **pricing and availability**.
- **App Review information**: contact details, and a **demo account** if your app requires sign-in.

:::warning Privacy is taken seriously
Apple requires a **privacy manifest** (`PrivacyInfo.xcprivacy`) declaring your use of certain APIs (like UserDefaults or file timestamps) and any tracking domains. Xcode's app template and most SDKs include theirs; check **File → New → File → App Privacy** if you need one. Apps that track users across other companies' apps must ask permission with App Tracking Transparency.
:::

## App Review

Select your build on the version page and click **Add for Review**, then **Submit**. Most apps are reviewed within a day or two. Common rejection reasons:

- Crashes or obvious bugs (test on a real device!).
- Placeholder content, broken links or unfinished features.
- Missing or vague **usage descriptions**, or asking for permissions you don't need.
- Missing privacy policy, or inaccurate privacy details.
- Login required but no demo account provided.
- Apps that are just a website in a wrapper, with no native value.

Read Apple's **[App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)** before your first submission. If you're rejected, the message explains why; fix it and resubmit, or reply in Resolution Center.

## Releasing

Choose how the approved version is released: **manually**, **automatically** after approval, or on a date. **Phased release** rolls an update out to automatic-update users over 7 days, and you can pause it if a problem appears.

Then monitor **Organizer → Crashes** and **App Store Connect → Analytics / Ratings and Reviews**, and reply to reviews.

:::note Staying current
Each spring Apple requires new uploads to be built with a recent Xcode and SDK. Keep Xcode up to date and ship an update at least once a year to stay eligible.
:::

## Updates

For every update: bump the **version** (and build), archive, upload, test in TestFlight, fill in **What's New**, and submit. Consider `StoreKit`'s `requestReview` to ask happy users for ratings at a good moment, such as after they've saved their third favourite.

## Recap

:::recap
- Final bundle ID, polished icon, tested edge cases and a privacy policy before shipping.
- Paid Developer Program + automatic signing = Xcode manages certificates and profiles.
- Unique, increasing **build** numbers; **version** is user-facing.
- Product → Archive → Distribute → App Store Connect.
- TestFlight internal and external testing, then the listing, privacy details and App Review.
:::

:::exercise
1. Write a draft App Store listing for RecipeBox: name, subtitle (30 characters), promotional text, a short description, and 10 keywords.
2. List the data RecipeBox collects for the App Privacy section. (Hint: if favourites stay on the device and you have no analytics, the answer may be "Data Not Collected".)
3. If you have a paid membership: archive one of your practice apps and send it to yourself via TestFlight.
:::

## Check your understanding

```quiz
Q: What must change for every build you upload to App Store Connect for the same version?
- [ ] The bundle identifier
- [x] The build number (it must be unique and increasing)
- [ ] The app name
- [ ] The signing certificate
> Build numbers identify each upload.

Q: What does "Automatically manage signing" do?
- [ ] Skips code signing
- [x] Lets Xcode create and renew certificates and provisioning profiles for you
- [ ] Submits the app for review
- [ ] Encrypts the app
> Automatic signing removes most manual portal work.

Q: How do you create the build you upload to App Store Connect?
- [ ] Copy the .app from the Simulator
- [x] Product → Archive, then Distribute App from the Organizer
- [ ] Email the project folder to Apple
- [ ] Build for Simulator in Release mode
> Archives are signed, optimised release builds.

Q: Which testers can install a TestFlight build without any review?
- [x] Internal testers on your App Store Connect team
- [ ] External testers with a public link
- [ ] Anyone with the App Store
- [ ] Nobody
> External testing requires Beta App Review for the first build of a version.
```
