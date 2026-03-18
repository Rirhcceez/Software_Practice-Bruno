# 🍜 Auntie Som’s Noodle Stall API - QA Audit Report

## 📌 Overview
This repository contains an automated API testing suite built with **Bruno**. The objective of this project is to audit and validate the backend API of "Auntie Som's Noodle Stall" (developed by Nephew Lek) to ensure data integrity, security, and correct business logic before production deployment.

## 🛠️ Tools Used
* **API Client & Testing Framework:** Bruno
* **Test Strategy:** Negative Testing, Business Logic Validation, and Data Dependency Chaining.

---

## 🐞 Bug Summary & Detection Strategy

During the automated testing process, **3 critical bugs** were discovered. Below is the summary of each vulnerability and how the Bruno test scripts detect them.

### Bug 1: Negative Quantity Injection (Critical Security/Logic Flaw)
* **Description:** The `POST /orders` endpoint does not validate if the `quantity` field is a positive integer. Users can send negative values (e.g., `"quantity": -3`).
* **Impact:** This causes the `totalPrice` to become negative (essentially paying the customer) and mathematically *increases* the stall's inventory (`stock -= (-3)`).
* **How the test detects it:** * **Test Script:** `01_Send negative quantity`
  * **Method:** Sent an order payload with a negative quantity. The assertion expects `res.status eq 400` (Bad Request). 
  * **Result:** The test **FAILS** because the API improperly accepts the payload and returns `200 OK`. A subsequent test (`02_Get_stock_update`) verifies that the stock remains unchanged, which also fails as the API artificially inflates the stock count.

### Bug 2: Unrestricted Stock Ordering (Business Logic Bypass)
* **Description:** The API checks if an item is completely out of stock (`stock <= 0`), but fails to verify if the requested order `quantity` exceeds the currently available `stock`.
* **Impact:** Customers can order 10 bowls of Tom Yum even if only 2 are available, resulting in negative stock levels and unfulfillable orders.
* **How the test detects it:**
  * **Test Script:** `02_Send_order_quantity_that_exceed_stock`
  * **Method:** Sent a valid request with a quantity greater than the existing inventory. The assertion strictly expects `res.status eq 400` (Out of stock/Bad Request).
  * **Result:** The test **FAILS** because the API returns `200 OK` and allows the order to pass through, pushing the database stock into negative numbers.

### Bug 3: Unauthorized Price Deduction (Calculation Flaw)
* **Description:** The backend calculation contains a hardcoded `- 5` deduction on every order (`total_price = (item["price"] * quantity) - 5`). 
* **Impact:** Every transaction is undercharged by 5 Baht, leading to continuous revenue loss for the business.
* **How the test detects it:**
  * **Test Script:** `01_Send_a_order` & `02_Get order details`
  * **Method:** Data dependency chaining is used here. A standard order is successfully created, and its `orderId` is saved as an environment variable. The second script retrieves this specific order and asserts that the `totalPrice` exactly matches the expected mathematical calculation (e.g., 1 bowl * 50 Baht = 50).
  * **Result:** The test **FAILS** because the assertion `totalPrice eq 50` mismatches the API's returned value of `45`.

---

## 🚀 Conclusion
The current iteration of the API is **NOT ready for production**. The lack of server-side data validation (trusting client input) and incorrect mathematical logic poses a severe risk to the business. It is highly recommended that the development team implement strict payload validation and fix the pricing logic before any real-world deployment.