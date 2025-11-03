---
title: Introduction
sidebar_position: 0
---

# Verifier


Multipaz Identity Verifier( Multipaz Identity Reader) allows you to request and display identity credentials from another person using QR codes, NFC, or Bluetooth Low Energy (BLE), in compliance with ISO/IEC 18013-5:2021. It works entirely offline, without requiring an Internet connection, and supports importing IACA certificates and VICALs. we will only be using MpzIdentityReader for verification of this codelab.

The source code for the Multipaz Identity Reader is available **[here](https://github.com/openwallet-foundation/multipaz-samples/tree/main/MultipazCodelab/Reader)**. Please build the apk by yourself.

When Holder app and Verifier app starts to communicate, they will  verify each other’s certificate to make sure they are in the trusted list.

In the holder app, we already talked about how to add Verifier’s certificate,here we will talk about Verifier app Holder:  turnning on ble ,verification mode and adding  certificate.