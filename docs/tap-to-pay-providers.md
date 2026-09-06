# Tap to Pay on iPhone — Payment Provider Comparison

Goal: from the loyalty POS cash register, click **Pay by card** → the amount is pushed to an iPhone, which acts as the card reader (Tap to Pay on iPhone). Customer taps their card/phone on the iPhone; result is reported back to the POS.

## The 3 candidates

| | Stripe Terminal | Viva.com | Adyen |
|---|---|---|---|
| How it works | Small companion iOS app built with the Stripe Terminal iOS SDK; POS backend creates a PaymentIntent, iPhone app collects it via Tap to Pay | **No app to build** — install the free Viva.com Terminal app on the iPhone; POS calls the Cloud Terminal REST API (or local ECR API) to push the amount to the phone | Companion iOS app built with the Adyen POS Mobile SDK; POS backend sends a Terminal API payment request routed to the iPhone |
| iOS development needed | Yes (small app; official sample app available) | No | Yes |
| Apple Tap to Pay entitlement | You request it for your app | Not needed (Viva's app has it) | You request it (TEST + LIVE, LIVE can take weeks) |
| Countries | US, UK, CA, AU + much of EU | ~22 European countries + UK (no US) | US, UK, EU, AU + more (enterprise onboarding) |
| Test/sandbox | Excellent — test mode with simulated reader | Demo environment + demo app | Test environment, but sales-led onboarding |
| Setup speed for a test | Days (app + entitlement) | Hours | Weeks |
| Indicative pricing | ~2.7% + $0.05 per in-person charge (US; check current) | ~1.2–1.5% EU cards (varies by country/volume) | Interchange++ (contract, aimed at large volume) |

## Recommendation

1. **Fastest way to test (EU/UK): Viva.com.** Zero iOS coding — the POS just POSTs `{amount, sessionId}` to the Cloud Terminal API and the Viva.com Terminal app on the iPhone prompts "Tap card". Exactly the ECR → phone-as-terminal flow.
2. **Best for a custom loyalty POS, and the US option: Stripe Terminal.** Build a minimal iPhone "reader" app from Stripe's sample; the web POS creates the PaymentIntent and the phone collects it. Best docs, webhooks, and test mode.
3. **Adyen** only if this will scale to enterprise/multi-country later; heaviest onboarding for a quick test.

## Flow (all three)

```
POS "Pay by card" → POS backend creates payment (amount, ref)
                  → pushed to iPhone (provider cloud API or your own push/socket)
                  → iPhone shows "Tap to Pay" → customer taps card
                  → provider webhook → POS marks sale paid
```

Note: Tap to Pay on iPhone requires iPhone XS or newer on a recent iOS, and only works in Apple-supported countries.

## References

- Stripe: https://stripe.com/terminal/tap-to-pay-on-iphone and https://docs.stripe.com/terminal
- Viva.com Cloud Terminal API: https://developer.viva.com/apis-for-point-of-sale/card-terminals-devices/rest-api/
- Adyen: https://docs.adyen.com/point-of-sale/mobile-ios/build/tap-to-pay
- Apple country list: https://developer.apple.com/tap-to-pay/regions/
