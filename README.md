# bean-acb

`bean-acb` is a Python script that takes a Beancount ledger and a config file
and outputs the computed Adjusted Cost Base (ACB) and capital gains that should
be declared for a given property.

Note that you should *not* rely on this script to declare your taxes, there is
no test suite and the implementation is my best understanding of the tax rules
but it might have errors.

## Why bean-acb?

When selling property, capital gains must be computed based on the
[adjusted cost base](https://www.canada.ca/en/revenue-agency/services/tax/individuals/topics/about-your-tax-return/tax-return/completing-a-tax-return/personal-income/line-12700-capital-gains/calculating-reporting-your-capital-gains-losses/adjusted-cost-base.html)
of the property.

Usually, transactions would appear like so in a ledger:

```
2025-01-01 * "Buying some HOOL"
  Assets:US:ETrade:HOOL                10 HOOL {160.00 USD}
  Assets:US:ETrade:Cash          -1609.95 USD
  Expenses:Financial:Commissions     9.95 USD

2025-01-02 * "Buying more HOOL"
  Assets:US:ETrade:HOOL                 5 HOOL {170.00 USD}
  Assets:US:ETrade:Cash           -859.95 USD
  Expenses:Financial:Commissions     9.95 USD

2025-01-02 *"Selling HOOL"
  Assets:US:ETrade:HOOL                -3 HOOL {160.00 USD} @ 180.00 USD
  Assets:US:ETrade:Cash            530.05 USD
  Expenses:Financial:Commissions     9.95 USD
  Income:US:ETrade:PnL              60.00 USD
```

In some jurisdictions, declaring 60.00 USD (or 50.05 if deducting the
commission) would be enough, but according to Canadian rule, the capital gain
to declare is not `60.00 USD`.

Instead, we must first compute the total acquisition cost:
> 10 * 160.00 + 9.95 + 5 * 170.00 + 9.95 = 2469.90 USD

The ACB per share is:
> 2469.90 / 15 = 164.66 USD

Total proceeds from the sell, from which we deduce the commission:
> 3 * 180.00 - 9.95 = 530.05 USD

And finally we can compute the capital gain:
> 530.05 - 164.66 * 3 = 36.07 USD

The total ACB is also reduced by `164.66 * 3` and becomes:
> 2469.90 - 164.66 * 3 = 1975.92 USD

Which divided by the remaining shares gives the same ACB/share as before:
> 1975.92 / (10+5-3) = 164.66 USD

Usually, ACB/share only changes when share are bought or dividend reinvested.

Note that these amounts are in USD and should be converted using the exchange
rate of the day on which transactions occur. So the whole ACB computation might
get cumbersome very fast, hence the need for bean-acb.

## How does this script work?

It takes a ledger file and a config file as parameters. The config file looks
like this:

```json
{
  "HOOL": {
    "investments": [
      "Assets:US:Etrade"
    ],
    "commissions": [
      "Expenses:Financial:Commissions"
    ]
  }
}
```

It looks for transactions in accounts whose name start with one of the entry in
the `investments` list, and categorizes them as buy or sell event depending on
the sign of the transaction. For buy event it looks at the cost and for sell
event at the price. Additionally, commissions can be taken into account but
accounts also have to be listed explicitly.

As the ACB must be expressed in Canadian Dollars (CAD), the script will convert
all other currencies to CAD and warn if the price date for that date is not
present in ledger, as this is also a requirement of the tax rule.

## Superficial Loss

Not all capital losses can be declared, as there is a rule called
[superficial loss](https://www.canada.ca/en/revenue-agency/services/tax/individuals/topics/about-your-tax-return/tax-return/completing-a-tax-return/personal-income/line-12700-capital-gains/capital-losses-deductions/what-a-superficial-loss.html).
Technically, the rule also applies to *identical* property, but that cannot be
determined automatically by the script and is not supported, so only the *same*
property is looked for to determine if loss is superficial or not.
