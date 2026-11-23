# Cousin Ornament Exchange

Every year my extended family runs a Cousin Ornament Exchange (COE): each participant gives one ornament to one other participant. It is a secret Santa, with two extra rules:

- no one is paired with a spouse or a sibling
- no one gets the same recipient twice across years

This repo holds the two iterations of the software I built to run it.

| Version | Folder | What it does |
| --- | --- | --- |
| Website | [`website/`](website/) | ASP.NET MVC site where family members registered and I tracked responses |
| Console app | [`console-app/`](console-app/) | Computes the pairings, sends the emails, and enforces the rules above |

The website was replaced by a Google Form for gathering responses, and the console app took over everything else. The git history interleaves both, so `git log` shows how the project evolved.

## Console app

A .NET 8 console app plus NUnit tests.

- `Parsing.cs` and `Calculations.cs`: build the compatibility matrix and generate pairings (see below).
- `Email.cs`: sends the invitation, the "wall of fame", the "wall of shame", and the final pairings.
- `Parsing.cs` also reads the responses exported from the form.
- `COE.Tests/`: validates the family data (spouses, siblings, complete history, no repeat recipients).

```bash
cd console-app
dotnet test
```

### Pairing algorithm

The pairing is a traveling salesman problem in disguise: one tour that visits everyone exactly once, where each person gives to the next person on the tour and the last gives to the first.

1. **Cost matrix.** `Parsing.GetCompatibilityMatrix` gives every (giver, receiver) pair a penalty. A spouse costs 50, a sibling 25, and a past gift costs `year - 2001`, so recent years weigh more. A cost of 0 is a pair that is allowed.
2. **Greedy tour.** `Calculations.GetPairings` is a nearest-neighbor heuristic. Starting from the first participant, it repeatedly moves to the unvisited person with the lowest cost from the current one. Ties go to the most constrained person (the one with the highest total penalty), so the hardest-to-place people are used early.
3. **Forced matches.** `Program.ForcedMatches` pins specific giver-to-receiver pairs and removes those receivers from the greedy search.
4. **Random restarts.** A greedy tour can paint itself into a corner. `Email.SendPairings` accepts a result only if its total cost is 0, meaning no spouse, no sibling and no repeat from any past year. Otherwise it reshuffles the participants with a seeded `Random` and tries again.

This is a heuristic, not an exact solver: it can fail on a given ordering, but the restart loop makes it practical for a family-sized group.

**This code does not build as published.** The family data (`Data.Family` and the `Name` enum) is private and was never committed, so you would need to supply your own. Email sending reads the Gmail password from the `GmailPassword` app setting, and the sender address in `Email.cs` is a placeholder.

## Website

An ASP.NET MVC 3 site (.NET 4, SQL Server Compact) that handled registrations, relationships, and pairing history. Open `website/robkeim.sln` in Visual Studio to run it. Personal data about my family was removed before publishing, so the seed data is empty.
