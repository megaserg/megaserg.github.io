---
layout: post
title: "What is compensation made of"
tags: compensation, money, offers
crosspost_to_medium: true
---

**Disclaimer:** This post is about software engineer compensation in United States. I'm using **40%** total tax rate in calculations. I am also not a lawyer or tax advisor.

Sometimes you find yourself in the situation where you have multiple job offers and need to choose between them. There are many variables that should come into your decision, and one of them is comparing the financial side of the offers. Some of the components are not obvious, so this post will hopefully help you to evaluate the total fair market value of your offers. Make sure to ask your contacts at the company about these items, if they are not explicitly stated in writing.

# Basics

- **Base yearly salary.** Pretty clear. Usually paid every other week. For Bay Area, varies within **$100k-200k** for most people, depending on experience. Taxed at about **38%** (**22%** federal tax + **8%** California tax + **6%** Social Security tax + **2%*** other taxes).

- **Equity: RSU/options.** There are [multiple](https://github.com/jlevy/og-equity-compensation) [articles](https://gist.github.com/yossorion/4965df74fd6da6cdc280ec57e83a202d) explaining how to evaluate equity, so I won't go into details. RSU profits are taxed a bit higher than cash salary. Usually equity is vesting over four years, so total yearly compensation is calculated as `base_salary + equity_grant / 4`. However, there are other components in the equation.

# Bonuses

- **Signing bonus.** One-time taxable bonus, usually is given with the first paycheck. Can be as high as **$100,000** (e.g., for returning interns at Facebook). If you leave the company before your first anniversary, you would be asked to return it pro-rated for the rest of the year. Can be easier to negotiate than the base salary: for the company it’s a one-time expense, as opposed to a recurring one.
- **Relocation bonus.** Also taxable and usually given with the first paycheck (which means you will need to spend your own money until then). Can also be non-cash: the company might pay for the ticket and provide accommodation e.g. for the first month.
- **Annual bonus policy.** Profitable companies often have a “target” annual bonus which can be **10%** or **15%** of the base salary, and the actual bonus for the year is somehow computed from the company performance and your personal performance.
- **Raises/promotion policy.** Usually the companies at least try to keep up with inflation, so you can expect **1-2%** raise yearly. However, you should also keep in mind whether it's going to be easy to get promoted (and get a raise/bonus associated with it).
- **Performance/per-project bonus policy.** Some companies reward engineers based on evidence they've put in good work: after a performance review, or after a release was shipped.

# Other perks

- **ESPP.** You can set off some post-tax money from your paycheck to buy your company stock at a discount, e.g. **15%**. The amount that can go towards the purchase is limited by the Purchase Plan at some (e.g. again **15%**) of your base salary, and by IRS at **$25,000** per calendar year - whichever is lower - so the minimum profit (if you sell immediately) is `0.15 x min(25000, <base>x0.15)`, which is **2.25%** of your base or **$3,750**, whichever is smaller. The better the discount and the better your stock is doing, the better your profit.
- **PTO/sick leave policy.** There are **50** full business weeks in a year. If two companies offer the same base pay, but company A offers **15 days (3 weeks)** of vacation, and company B offers **20 days (4 weeks)**, that extra week is equivalent to getting **1/50 = 2%** of the base pay without having to work for it. You basically get paid extra **2%** for every extra vacation week over baseline. Be cautious about the "unlimited PTO" policy: as you can guess, it's not *really* unlimited - either you come up with a guilt-induced limit for yourself, or you hit the informal unspoken limit, beyond which your manager is not comfortable with your absence.
- **401(k) matching.** Some companies encourage you to set aside a portion of your paycheck (pre-tax) to save for your retirement, by matching your contribution to your retirement account. If the company has this program (not just *401(k)* program, but *401(k) matching* program), then for every dollar you contribute, the company will contribute **50** cents (if they match **50%**) or a dollar (if they match **100%**). There is a limit set by government on how much you can contribute per year (**$18,500** as of 2018). Additionally, the company may have a limit of how much they will contribute (e.g. **$1,500**/year at Twitter). The company's one-time contribution happens after the year has ended, e.g. in mid-February of the next year.

# Reimbursements and subsidies

- **Free/subsidized food.** In Bay Area, this is easily worth **$20-$30** per meal, **$50-60** with dinners. **$25** daily (post-tax) = **$10500**/year (pre-tax).
- **Shuttles/commuter subsidy/Uber credits.** Depending on how you commute, every **$12** (post-tax) spent on a daily commute are worth **$5000**/year (pre-tax).
- **Reimbursement policy:** books, conferences, travel, offsites. This could be easily worth thousands per year.
- **Gym reimbursement.** In Bay Area, this can cost **$50-150** per month (post-tax), so gym reimbursement is equivalent to **$600-1800**/year (post-tax) or **$1000-3000**/year (pre-tax).
- **Insurance premiums** for you and your family. This is **$200-300** per month = **$2400-3600**/year. However, almost every company takes care of that for you, so it's unlikely to be a decision factor.

# Other concerns

If you're choosing between different states, take into account:
- **state taxes** (varies [from **0%** to **13%**](https://en.wikipedia.org/wiki/State_income_tax))
- **cost of living** (especially rent - expect to spend $2000-$3000/month (post-tax) = **$40k-60k** (pre-tax) in Bay Area)
- **sales tax** (varies [within **6-9%**](https://taxfoundation.org/state-and-local-sales-tax-rates-in-2017/))

This should help you to put a price tag on many perks and benefits that the companies do or don't provide. Remember that it may be short-sighted to compare companies from the financial side only, and also to evaluate offers based on the current stock price or current startup valuation. Who knows what will happen in a couple years? Good luck!
