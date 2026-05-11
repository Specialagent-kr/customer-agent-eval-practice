# LLM-as-Judge Evaluator Design Guide

This guide explains how to build binary Pass/Fail evaluators for subjective assessment criteria that code-based checks cannot handle.

## When to Use LLM-as-Judge

Use judges when "a failure mode requires interpretation (tone, faithfulness, relevance, completeness)"
but avoid them if "the failure mode can be checked with code (regex, schema validation, execution tests)."

## Core Requirements

Before designing a judge, you need:
- Completed error analysis identifying the specific failure mode
- At least 20 Pass and 20 Fail human-labeled examples
- Confirmation that code-based solutions won't work

The guide notes that seemingly subjective problems often have simpler solutions—for instance,
detecting vague interview questions "seems to require semantic understanding, but...a keyword check
for words like 'usually,' 'typical,' and 'normally' could work quite well."

## Four Essential Components

Each judge prompt must include:

1. **Task Definition**: State exactly what one criterion you're evaluating
2. **Pass/Fail Definitions**: Binary outcomes only—"define exactly what constitutes Pass and Fail"
3. **Few-Shot Examples**: Include clear Pass, clear Fail, and borderline cases from your training data
4. **Structured Output**: Use schema enforcement requiring both a detailed critique and verdict

## Critical Anti-Patterns to Avoid

- Using vague criteria like "is this helpful?"
- Building holistic judges covering multiple dimensions
- Using Likert scales, which "sound precise but can't be calibrated"
- Leaking dev/test data into few-shot examples
- Skipping validation against human labels
