---
layout: page
title: Statistical Arbitrage in a Niche Market
description: A survival-analysis approach to arbitrage in a slow, data-starved, decentralized market, co-founded, modeled, and deployed with real money.
img: assets/img/arbitrage_sde_forecast.jpeg
importance: 1
category: work
related_publications: false
---

Classical statistical arbitrage assumes two things this market does not have: speed and clean data. The textbook version runs at high frequency over deep, well-behaved order books. We wanted to know whether the same ideas survive once you take both of those away, so we built a framework for a niche, decentralized in-game-item market with slow, *enforced* settlement times, sparse history, and a free data API that was actively being deprecated underneath us.

This was a real project with real money in it, co-founded with Lucas Logue and run from March 2025 onward. We deployed $13,500 of our own capital and traded it live for about 5.7 months. The honest version of what happened (including the part where the market changed the rules overnight and the model fell over) is below. The full paper goes much deeper.

*Co-founded with Lucas Logue · March 2025 – present · Python (hierarchical Bayesian modeling), survival analysis, stochastic differential equations, custom data pipeline.*

## Why survival analysis

The non-obvious choice here is the modeling frame. When you buy an item in this market, you are forced to hold it for seven days before you can trade it again. That hold is the whole problem. The risk isn't really price. It's *time*: after seven days the item may have lost value, the arbitrage window you bought into may have closed, and you're left holding something worth less than you paid.

That is a survival problem. Instead of asking "what will the price be," we ask "how long does an arbitrage opportunity stay open, and what's the probability it's still open by the time my hold expires." The model picks the items most likely to survive the wait. Framing it as duration rather than price is what made the rest of the math tractable.

We built this as a **hierarchical Bayesian Weibull Accelerated Failure Time (AFT)** model. The Weibull AFT's additive log-duration structure held up well under sparse, messy data, which mattered more than usual here. Two custom features we engineered for this market, a congestion metric and a benchmark-ratio deviation, came out statistically significant (94% HDI excluding zero), which was a relief, because we made them up.

<div class="row justify-content-sm-center">
    <div class="col-sm-7 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/arbitrage_survival_curves.jpeg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Posterior survival curves: the modeled probability that an arbitrage opportunity is still open over time, with credible bands. Each panel is a different asset category. The model's job is to pick items whose curve stays high through the seven-day hold.
</div>

To make sure we weren't just believing our own model, I built a separate **Bayesian hierarchical piecewise-exponential Cox proportional-hazards model** to benchmark the Weibull AFT retrospectively. The idea was to attack our own work from a different structural assumption and see whether the conclusions held. They mostly did, but the Cox model also showed its limits: an optimally-parameterized Cox PH got corrupted by the market's data sparsity in a way the AFT didn't, and the AFT came out about 20% more accurate. The lesson was the useful kind: the "better" model on paper was the worse model for *this* data.

## Forecasting price with a jump-diffusion SDE

Survival picks *what* to buy; we still needed a view on *price* to size and time trades. Standard Geometric Brownian Motion is a bad fit for this market. It assumes a kind of smooth, continuous drift that simply isn't how these items move. We used a **jump-diffusion stochastic differential equation** with a Bayesian-learned, structurally-specified drift term to forecast commodity prices, so the model could express both the slow grind and the sudden jumps.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/arbitrage_sde_forecast.jpeg" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Custom SDE price forecast for one commodity: historical price (black), median forecast (dashed), and the probabilistic fan of simulated diffusion paths. The widening cone is the model being honest about how little it knows past a certain horizon.
</div>

## The data pipeline

This was my main piece of the early work. The market's data lived behind a free, soon-to-be-deprecated API with tight rate limits, so naive fetching was a non-starter. I built a batch-fetching pipeline that pulled the daily dataset we needed in roughly 150 calls at an 8:1 compression ratio, enough headroom to keep the models fed without tripping the limits or depending on an API that might vanish. Sparse markets punish you twice: once for having little data, and again for making it expensive to get. The pipeline was the difference between a model we could actually run daily and a nice idea.

## The part where it broke

Phase 3, somewhere in the September–November 2025 window, was a genuine market crash, and it's the part I'd rather not skip. The platform operator changed a core game mechanic in a way that was about to flood the supply of an entire category of high-value items. Almost everything moved at once: every open arbitrage window closed simultaneously, and that whole category instantly lost 30%+ of its value on the anticipated future supply.

The model did not hold. It was never built for a structural regime shift of that size. It had been quietly assuming the rules of the world were fixed, which is exactly the assumption that just broke. The fix wasn't a parameter tweak; we went back and modified the model to explicitly account for major market shifts, so that pre-shift data couldn't corrupt the current estimates. A model that has never seen a regime change will trust the old world right up until it loses you money. Now it doesn't.

## Did it work

Over the live run, by our most conservative bookkeeping (the gains we're willing to attribute specifically to the models, after platform fees), $13,500 became about $19,100, a return of roughly 40% in 5.7 months. For reference, the S&P 500 returned about 8% over the same window. Max drawdown was around 38%, mostly courtesy of Phase 3. We succeeded on more than 70% of arbitrage trades, and the portfolio beat the broad niche-market index it was trading inside by a wide margin.

The number I actually care about is calibration. When the model said a bracket of trades had a 40–50% chance of working, the realized rate came in around 54%, i.e. it was conservative, under-promising and over-delivering, which is the failure mode you want if you have to pick one. A trading model that's wrong in the optimistic direction loses real money; ours leaned the other way.

## Who did what

Lucas did most of the original modeling. My share was the data pipeline and a good chunk of the live trading, and I later built the Cox proportional-hazards benchmark to stress-test the original models and the post-crash Phase 4 model, to find the weaknesses if there were any, and confirm the strengths if there weren't.

On the real-money part: it meant we couldn't afford to be sloppy with the model, because a real mistake meant real losses. We put our own savings behind it because we believed the idea was right and wanted to prove it with skin in the game rather than a backtest. The paper's acknowledgments thank one author for "entrusting the other with his savings," which is funnier in hindsight than it was at the time.

<hr>

[**View the full writeup (PDF)**]({{ '/assets/pdf/statistical_arbitrage_paper.pdf' | relative_url }}), the complete ~50-page paper, co-authored with Lucas Logue, with full model specifications, priors, posterior tables, and SDE derivations.
