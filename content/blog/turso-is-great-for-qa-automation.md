---
title: Turso is Great for QA Automation
date: 2026-01-28
isCompleted: true
slug: turso-is-great-for-qa-automation
categories: [Web Dev]
tags: [turso, qa, automation, salesforce]
---

Once upon a time at work, I was tasked with writing automated tests for a web application built on top of Salesforce. The devs used a special plugin to implement a custom UI, but the underlying UI and Backend were still Salesforce. Now, I know what you're thinking. You shouldn't be testing Salesforce. That's Salesforce (the company)'s job. And you'd be right, but it's not just SF, it's our team's implementation of hundreds of little customizeable widgets, features, formulas, plugins and extras that all work in part to create a CMS.

Okay great. SF has a well-documented, robust API. Just setup Playwright to hit the sandbox environment that stays slightly ahead of prod and you're all set. Right? Well...

_Note: I won't actually mention Turso until [this part](#turso-for-tracking-test-data-for-cleanup), so check that out there._

## Hurdle 1: Authentication

I initially handled test user logins by disabling MFA for specific IPs. However, when I learned how many different IPs Github Actions servers can use, I knew that was only a temporary solution (not to mention how dynamic IPs would be changing all the time anyways). Instead, the better option turned out to be setting up TOTP for the test user's SF account, and then using a different NPM TOTP package to produce the time-based code, after storing the key.

This was further complicated when I wanted to only log in once per test cycle, and not once per test (waaaay too long). Fortunately, using a globalSetup() hook and context storageState, I was able to get this working too. I was able to store all the auth tokens in that storage state, and the other tests can start pre-authed from scratch. Nice.

## Hurdle 2: Tabs

The UI of Salesforce's lightning experience is robust, but also kind of clunky. One particular clunk is the feature of remembering what SF page you were looking at previously and storing it in a subtab. I'm not actually sure if this is SF's or my team's decision, but either way, it would probably be better to leave tab management to the browser, rather than rebuilding tabs inside the web app.

These tabs are very problematic for automated testing because not only do they introduce visual noise, SF stores each of those inactive tabs inside the DOM, presumably for quick reloading / swapping. This makes it even harder to target selectors, especially when multiple tests are run at the same time.

To avoid this shared state problem, I had to write a non-trivial setup script to close all the tabs first. This of course, adds execution time, but I couldn't find a way to do it through the API's recent object list.

## Hurdle 3: Cleaning up Test Data

Our instance of Salesforce was not lonely. Several other services, some internal and some third party, were integrated into the CMS and used various formulas and hooks to move data out and handle it. For example, entering a new "Car" in SF would automatically trigger the "Car Insurance" microservice to pull the owner's name and car model. These pulls were background jobs, and other teams were watching them for errors.

Unfortunately, this meant that cleaning up that test car could trigger errors in downstream services, and to avoid them, I was instructed to delay removing the objects at the end of the test. For context, I was running the test then immediately running cleanup. That was happening too fast. Delaying cleanup would allow the downstream processes to complete correctly, then handle object deletion correctly.

But once a test runs, that process data is lost unless you store it somewhere, and now we have a data storage and retrieval problem, on top of a test automation problem. Storing locally would not work since the tests run in a CI/CD pipeline. Storeing in-repo is equally bad for syncronization. That leaves us needing a lightweight, remote source-of-truth to track objects that need to be cleaned up.

Enter Turso.

## Turso for Tracking Test Data for Cleanup

As a service, Turso was a perfect fit. High availability and uptime, simple integration via npm package, good documentation and observability, and crucially, low-cost Sqlite dbs. It took me a day to get this part up and running, but once I did, the system worked brilliantly. It's great to be able to just open the Turso website and check how many objects need to be cleaned up.

In my setup, I use the SF API to pull all the objects created by my test user, then I order the objects by type and remove all of them older than 30 minutes old. Voila, no more downstream errors (that people are complaining about to me, at least).

## Turso for Other Things

In the past, I would have handled this problem in a worse way. Each testing project is unique in the constraints that you have, and that defines your range of options. But from now on I think Turso will be my default when I need to store test data outside of the system I'm actually testing. Believe me, I would prefer to just cleanup the data in SF and call it a day. For a persistent test environment that is used by other people, cleaning up after every test goes a long way toward atomicity. But given the requirements of this project, I'm happy a simple external data store exists like Turso.

The infrastructure Turso has created is extremely attractive, not just for testing, but I can see it being useful for other applications as well. Turso gives you hosting, security, configuration, debugging, the ability to read and edit your data, clean Sqlite schema updates (annoying IYKYK), backups, and yet still provides an in-process data store for blazing speed. The alternative would be me taking weeks to set all that up (correctly!) and missing my deadlines.

It really is a revelation.
