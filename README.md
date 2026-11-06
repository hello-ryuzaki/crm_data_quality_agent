# CRM Data Quality Agent (HubSpot)

My idea for the Bloomwell application: an agent that finds duplicate records, gaps and conflicting fields in a CRM, and cleans them up with a person approving every change.

**Status:** this is a design and a proposal right now. I am building a small prototype on sample data, and I will update this page as parts get done. Nothing here uses real company or patient data, and it never will.

## The problem

Any CRM that has been running for a few years collects dirty data. The same contact gets created twice, once from a form and once from a manual entry. A company shows up with two different domains. Phone numbers sit in five formats. Nobody planned this, it just happens, and then support and operations teams waste time on the wrong record, or send two emails to the same person.

Cleaning it by hand is boring work, so it keeps getting postponed. That is why I think it is a good candidate for automation. But a CRM is also the place where a wrong merge hurts, so I do not want a tool that fixes things silently.

## What the agent does

1. Pulls contacts and companies (through the HubSpot API in a real setup, from a CSV in this repo).
2. Cleans the fields first: lowercase emails, standard phone format, company names without "GmbH", "Ltd" and similar endings.
3. Creates candidate pairs, using simple blocking (same email domain, same postal code, similar first letters), so we do not compare every record with every other one.
4. Scores each pair with fuzzy matching (rapidfuzz).
5. Pairs with a very high score are marked "likely duplicate". Pairs with a very low score are dropped. The pairs in the middle go to an LLM, which gives a structured answer: same entity or not, and a short reason.
6. Separately, it lists gaps (for example a company with no domain) and conflicts (the same company with two different addresses).
7. Everything goes into a review queue. A person looks at each batch and approves or rejects it.

## Human in the loop

Nothing is merged or changed on its own. The agent only proposes. Once a person approves a batch, the change is applied and the before and after values are written to an audit log, so any change can be reversed later. If the LLM is unsure, it says so, and the pair goes to the person with no suggested answer.

I like this design because the team can trust it from day one. They are not asked to believe the model, they are asked to check it, which is a much easier thing to agree to.

## How I would measure it

I would not guess whether the LLM step is worth it. The plan is:

- Make a small labelled set of fake records with known duplicates (I will generate these, no real data).
- Run fuzzy matching alone first and note precision and recall.
- Then add the LLM for the middle pairs and check if the numbers really improve, and by how much. If they do not improve, the LLM step goes out. It adds cost and delay, so it should earn its place.

In a live setup, the numbers I would track are:

- duplicates found, and how many the reviewers confirmed (this gives precision from real usage)
- review time per batch
- estimated hours saved per week
- LLM cost per 1,000 pairs

## Data protection

Bloomwell works with sensitive data, so this part matters more than the clever part. My approach would be:

- Send the model only the fields needed to compare two records (name, email domain, city), not the full record.
- Mask or hash personal details where the comparison still works.
- Keep health-related fields out of this process entirely.
- Use a model hosted in the EU, and keep access tokens in a secrets manager, not in code.

I would want to check all of this with the data protection team before anything touches real records.

## Rough design

| Part | Choice | Why |
|---|---|---|
| Language | Python | It is what I use daily |
| Fuzzy matching | rapidfuzz | Fast, simple, no setup |
| LLM step | Any model with structured output, validated with Pydantic | The answer must be a fixed shape, not free text |
| Storage | SQLite for the prototype, PostgreSQL later | Review queue and audit log |
| Review screen | Simple Streamlit page | Quick to build, enough for a first version |
| CRM connection | HubSpot API (later) | CSV for now |

## Planned repo structure

```
crm-data-quality-agent/
  data/            fake sample contacts and companies
  src/
    clean.py       field cleaning
    match.py       blocking and fuzzy scoring
    judge.py       LLM step for the middle pairs
    review.py      review queue and audit log
  eval/            labelled pairs and the evaluation script
  app.py           review screen
  README.md
```

## What is done and what is next

- [x] Idea and design (this page)
- [ ] Fake sample data with known duplicates
- [ ] Cleaning and fuzzy matching, with the first precision and recall numbers
- [ ] LLM step and the comparison with fuzzy matching alone
- [ ] Review screen and audit log
- [ ] HubSpot API connection

## Author

Sai Eswar Gujjala
[LinkedIn](https://linkedin.com/in/sai-eswar-gujjala) | [GitHub](https://github.com/hello-ryuzaki)
