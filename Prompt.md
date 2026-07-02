## Part A  Prompting for test design

This is the prompt I would give an AI tool:

I am testing a new checkout feature for an ecommerce site.

Partial user story:
"As a customer, I want to update my delivery address during checkout so that my order is delivered to the correct location."

Context:
- The feature is needed because wrong-delivery complaints are increasing.
- Developers say it is mostly done, but validation rules are still being finalised.
- Postcode rules may differ by country, but this is not confirmed.
- Current checkout already supports saved addresses, guest checkout and promotional discounts.
- Some users have multiple saved addresses.
- The order management system and confirmation email both use the delivery address after checkout.
- Test data is limited and shared across testers.
- There is no final RTM and acceptance criteria are still being refined.

Generate useful test scenarios and test cases for this feature.

Focus on:
1. Positive checkout scenarios
2. Boundary and validation scenarios
3. Negative address scenarios
4. Saved address and multiple-address scenarios
5. Guest checkout scenarios
6. Promo discount interaction
7. Order management system address propagation
8. Confirmation email address propagation
9. Country/postcode variation
10. Test data and environment risks

For each test case, include:
- Test objective
- Preconditions
- Test data needed
- Steps
- Expected result
- Priority or risk level
- Whether it is suitable for automation now or should remain manual/exploratory for now

Also highlight any missing requirements, assumptions, ambiguities, and questions that should be clarified before testing starts.
