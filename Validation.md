# Business logic and Validation

## FluentValidation 

Presiously the business validation was built a simple method Validate and rules based on a simple interface IRule.

FluentValidation does the same but in a better way and this the reason why the Guidance will use this.
</br>To ease the migration, compatibility with IRule is added.

See the official FluentValidation first: https://fluentvalidation.net/

### Validation 

Previously the code was:

```csharp

    public interface IRule
    {
        Messages Execute();
    }

    public interface IValidator
    {
        void Validate();
    }

    public abstract class ValidatorBase : IValidator
    {
        protected ValidatorBase(Messages messages)
        {
            _messages = messages;
            Rules = new List<IRule>();
        }

        protected IList<IRule> Rules { get; private set; }
        private readonly Messages _messages;

        protected void AddRule(IRule rule)
        {
            if (rule == null)
                throw new ArgumentNullException("rule");

            Rules.Add(rule);
        }

        public void Validate()
        {
            _messages.AddRange(Rules.SelectMany(rule => rule.Execute()));
        }
    }

```

A validation is a composition of rules and the Validate method is called to perform the validation. 
The result of the validation is simply given back via the Messages collection.

### FluentValidation

First a new package exists to integrate FluentValidation to Arc4u: Arc4u.Standard.FluentValidation

This nuget package (available from version 5.0.10.1) integrates FluentValidation with PersistEntity and Messages concept.


```csharp

    // No breaking change regarding the old rule design.
    public interface IRule
    {
        Messages Execute();
    }

    // No breaking change for the old validation pattern.
    public interface IValidator
    {
        void Validate();
    }

    public abstract class ValidatorBase<TElement> : AbstractValidator<TElement>, IValidator where TElement : class
    {
        protected ValidatorBase(Messages messages, TElement instance)
        {
            _messages = messages;
            Rules = new List<IRule>();
            _instance = instance;
        }

        protected ValidatorBase(Messages messages, ValidationContext<TElement> validationContext)
        {
            _messages = messages;
            Rules = new List<IRule>();
            _validationContext = validationContext;
        }

        protected IList<IRule> Rules { get; private set; }
        private readonly Messages _messages;
        private readonly TElement _instance;
        private readonly ValidationContext<TElement> _validationContext;

        protected void AddRule(IRule rule)
        {
            if (rule == null)
                throw new ArgumentNullException("rule");

            Rules.Add(rule);
        }

        public void Validate()
        {
            _messages.AddRange(Rules.SelectMany(rule => rule.Execute()));

            var result = null != _instance ? base.Validate(_instance) : base.Validate(_validationContext);

            _messages.AddRange(result.ToMessages());
        }

        public async Task ValidateAsync()
        {
            _messages.AddRange(Rules.SelectMany(rule => rule.Execute()));

            var result = (null != _instance) ? await ValidateAsync(_instance) : await ValidateAsync(_validationContext);

            _messages.AddRange(result.ToMessages());
        }
    }

```


The concept here is to proxy the FluentValidation so the old IValidator nd IRule concept can be reuse. This is essentially to
help the code migration.

One added value I see from the FluentValidation is the capability to build a Validation class where extra parameters can be added to the constructor 
to pass business logic interface needed to implement Must or MustAsync implementation!

#### Validation concept.

Like today by concept we implement we have information that we retrieve (Fetch of data) and save => insertion or update.

So each time a logic of a domain is implemented the code is generated with 3 basics validation: Get, Delete and Save (insert or update).

Often a Get is extremetly straightforward => checking that some rules are still respected.
Save and Delete will add more validations to ensure that the action can be performed.

We have then 2 aspects:
- the simple rule and this is where FluentValidation is great by reducing the code to write those basic settings.
- The complex ones to implement in a specific Validator with a Must or MustAsync method.

#### A basic validation.

```csharp

     internal class CompanySaveValidator : ValidatorBase<Company>
    {
        public CompanySaveValidator(Messages messages, ValidationContext<Company> validationContext)
            : base(messages, validationContext)
        {
            RuleFor(c => c.Id).NotEmpty();
            RuleFor(c => c.Name).NotNull().NotEmpty();
            RuleFor(c => c.PersistChange).IsInsert().Unless(ValidatorPredicates.IsUpdate);
            RuleFor(c => c.PersistChange).IsUpdate().Unless(ValidatorPredicates.IsInsert);
        }
    }

    // From the business logic code.

    await new CompanySaveValidator(errorList, entity).ValidateAsync().ConfigureAwait(false);


```

Basically we always design an object of the Domain based on a Arc4u.Data.PersistEntity! This govern the Bl and what we do
with the concept we are implementing => Save, Delete, Fetch...

#### Normal validation (meaning more than basic one)

In this case we will do simple validation and add more rules.

```csharp

    // A Rule to implement.

    internal class CheckUserCompanyExistRule : AbstractValidator<User>
    {
        public CheckUserCompanyExistRule(ICompanyBL companyBL)
        {
            _companyBL = companyBL;

            RuleFor(u => u.CompanyId).MustAsync(ExistAsync);
        }

        private readonly ICompanyBL _companyBL;

        private async Task<bool> ExistAsync(Guid id, CancellationToken cancellation)
        {
            if (id == Guid.Empty) return false;

            try
            {
                var company = await _companyBL.GetByIdAsync(id, cancellation);
                return null != company;
            }
            catch (Exception)
            {
                return false;
            }
        }
    }

    // A validator with a rule that will check the Company referenced in the User object exists!

    internal class UserSaveValidator : ValidatorBase<User>
    {
        public UserSaveValidator(Messages messages, User entity, ICompanyBL companyBL)
            : base(messages, entity)
        {
            RuleFor(c => c.Id).NotEmpty();
            RuleFor(c => c.Name).NotNull().NotEmpty();
            RuleFor(c => c.FirstName).NotNull().NotEmpty();
            RuleFor(c => c.LicenseKey).NotNull().NotEmpty();
            RuleFor(c => c.EMail).NotNull().NotEmpty().EmailAddress();
            RuleFor(c => c.CompanyId).NotEmpty();
            RuleFor(c => c.PersistChange).IsInsert().Unless(ValidatorPredicates.IsUpdate);
            RuleFor(c => c.PersistChange).IsUpdate().Unless(ValidatorPredicates.IsInsert);

            Include(new CheckUserCompanyExistRule(companyBL));
        }
    }

    // From the business logic code.

    await new UserSaveValidator(errorList, entity, companyBL).ValidateAsync().ConfigureAwait(false);

```

What is interesting to mention is that we provide the ICompanyBL implementation so the rule can use the bl to fetch the data!

*****************************************************************************************************************************
    Validation are rules that we implement to validate business aspects and so we have to stay at the business level! 
*****************************************************************************************************************************


#### ValidationContext

With ValidationContext we can provide to the validator a context that will contain some usefull extra behaviors I will say.

I see 2 important ones:
- The rules if needed.
- Some context Data we need => context.RootContextData["Key"] = Object;

##### RuleSet

We can at the level of a Validator build some building blocks that we can activate or not by providing in the context the validation rules (simple strings) we want apply.

```csharp

            RuleFor(c => c.Id).NotEmpty();
            RuleFor(c => c.Name).NotNull().NotEmpty();

            RuleSet("Concept1", () =>
            {
                RuleFor(c => ...
            });

            RuleSet("Concept2", () =>
            {
                RuleFor(c => ...
            });

```

By calling the validator with a context and rules will allow us to adapt the behavior of the validator based on the business needs.

```csharp

            var context = ValidationContext<Company>.CreateWithOptions(entity, options =>
                                                                                   {
                                                                                       options.IncludeRuleSets("concept1");
                                                                                       options.IncludeRulesNotInRuleSet();
                                                                                   });

            var errorList = new Messages(true);
            await new CompanySaveValidator(errorList, context).ValidateAsync().ConfigureAwait(false);

            return errorList;

```

We in this example will execute the rules no in a RuleSet {options.IncludeRulesNotInRuleSet()} and the specifi "concept1".

