# Enumerations Gem Analysis: Usage Patterns & Anti-Patterns

This repository contains analysis of the [enumerations gem](https://github.com/infinum/enumerations) for Ruby on Rails. I evaluated 9 different usage patterns found in code that's used in production.

## 📊 Quick Reference

| Pattern | Verdict |
|---------|------------|
| [Pattern 1: Simple Name Display](1-presenter-name.md)| Rails enum sufficient most of the times |
| [Pattern 2: Query Factories](2-query-factories.md) | Use service objects |
| [Pattern 3: Subset Filtering](3-enum-class-methods.md) | Constants work fine |
| [Pattern 4: Cross-Enum Mapping](4-cross-enum-mapping.md) | Use mapper classes |
| [Pattern 5: Rich Metadata](5-rich-metadata.md) | Strongest argument |
| [Pattern 6: Authorization Flags](6-authorization-flags.md) | Keep auth in policies |
| [Pattern 7: Class Factory](7-class-factory.md) | Use strategy pattern |
| [Pattern 8: Strong Parameters](8-strong-parameters.md) | Use form objects |
| [Pattern 9: Model Classes](9-pattern-abusement.md) | Avoid using it like this |

## Documentation

### Pattern Analysis Files
Each pattern is documented with:
- **Original Example** - Real code using the enumerations gem
- **Analysis** - What it does, pros/cons
- **Rails Enum Alternative** - Multiple options using standard Rails

## Recommendation

**Use Rails enum + value objects for the best of both worlds:**
- Performance of Rails enum
- "Everything in one place" benefit
- No gem dependency
- Standard Rails patterns

The enumerations gem solves a real problem, but you can solve it just as well with standard Rails patterns. The gem's main value is convenience, not capability. And that convenience comes with the cost of enabling anti-patterns that we've seen in production code.
