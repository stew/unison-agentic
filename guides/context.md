# Navigating the Authoritative Language Reference via MCP

When you need to verify exact Unison syntax, typing rules, or semantics, fetch them
directly from the official docs rather than relying on this repo.

## Source of Truth

The authoritative language reference lives in the `@unison/website` project under the
`docs.languageReference.*` namespace. Access it via MCP.

## How to Fetch Docs

1. `mcp__unison__search-definitions-by-name` with query `docs.languageReference`
   to list available topics.
2. Pick the specific topic term you need.
3. `mcp__unison__view-definitions` to read it.

Or search Share directly:
- `mcp__unison__share-project-search` on project `@unison/website`

## Language Reference Index

These terms exist in `@unison/website` on the `main` branch:

- `docs.languageReference.topLevelDeclaration`
- `docs.languageReference.termDeclarations`
- `docs.languageReference.typeSignatures`
- `docs.languageReference.termDefinition`
- `docs.languageReference.operatorDefinitions`
- `docs.languageReference.abilityDeclaration`
- `docs.languageReference.userDefinedDataTypes`
- `docs.languageReference.structuralTypes`
- `docs.languageReference.uniqueTypes`
- `docs.languageReference.recordType`
- `docs.languageReference.expressions`
- `docs.languageReference.basicLexicalForms`
- `docs.languageReference.identifiers`
- `docs.languageReference.nameResolutionAndTheEnvironment`
- `docs.languageReference.blocksAndStatements`
- `docs.languageReference.literals`
- `docs.languageReference.documentationLiterals`
- `docs.languageReference.escapeSequences`
- `docs.languageReference.comments`
- `docs.languageReference.typeAnnotations`
- `docs.languageReference.parenthesizedExpressions`
- `docs.languageReference.functionApplication`
- `docs.languageReference.syntacticPrecedenceOperatorsPrefixFunctionApplication`
- `docs.languageReference.booleanExpressions`
- `docs.languageReference.delayedComputations`
- `docs.languageReference.syntacticPrecedence`
- `docs.languageReference.destructuringBinds`
- `docs.languageReference.matchExpressionsAndPatternMatching`
- `docs.languageReference.blankPatterns`
- `docs.languageReference.literalPatterns`
- `docs.languageReference.variablePatterns`
- `docs.languageReference.asPatterns`
- `docs.languageReference.constructorPatterns`
- `docs.languageReference.listPatterns`
- `docs.languageReference.tuplePatterns`
- `docs.languageReference.abilityPatterns`
- `docs.languageReference.guardPatterns`
- `docs.languageReference.hashes`
- `docs.languageReference.types`
- `docs.languageReference.typeVariables`
- `docs.languageReference.polymorphicTypes`
- `docs.languageReference.scopedTypeVariables`
- `docs.languageReference.typeConstructors`
- `docs.languageReference.kindsOfTypes`
- `docs.languageReference.typeApplication`
- `docs.languageReference.functionTypes`
- `docs.languageReference.tupleTypes`
- `docs.languageReference.builtInTypes`
- `docs.languageReference.builtInTypeConstructors`
- `docs.languageReference.userDefinedTypes`
- `docs.languageReference.unit`
- `docs.languageReference.abilitiesAndAbilityHandlers`
- `docs.languageReference.abilitiesInFunctionTypes`
- `docs.languageReference.theTypecheckingRuleForAbilities`
- `docs.languageReference.userDefinedAbilities`
- `docs.languageReference.abilityHandlers`
- `docs.languageReference.patternMatchingOnAbilityConstructors`
- `docs.languageReference.useClauses`

## When to Use This

Use the MCP language reference when:
- You are unsure about exact syntax for an edge case
- The local guides give conflicting or unclear information
- You need to verify typing rules precisely
- You encounter a feature not covered in the guides here
