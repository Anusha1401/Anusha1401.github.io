# Submission

## Overview

From a systems perspective, this project involved building and maintaining integrations between multiple external AI and enterprise APIs — including Microsoft APIs and OpenAI's GPT models — while ensuring performance, cost control, and reliability in production.

---

## Backend Migration: Python → Ruby

A key part of this work was leading a full backend migration from a Python-based service to a Ruby-based service. This required:

- Careful **parity validation** across both services to ensure no regression in API behavior
- Improving long-term **maintainability** and aligning with the broader backend stack

---

## AI Integration & Prompt Engineering

On the AI integration side, the work involved direct use of OpenAI's API with a strong focus on prompt engineering and model behavior tuning in production.

### Prompt Design

- Designed prompts for **consistency and determinism** in production responses

### Model Parameter Tuning

| Parameter | Purpose |
|---|---|
| `temperature` | Controls randomness vs. determinism in outputs |
| `top_p` | Nucleus sampling — controls token probability mass |
| `max_tokens` | Constrains response length |
| `frequency_penalty` | Reduces repetition across the response |
| `presence_penalty` | Encourages diversity by penalizing already-used tokens |

### Model Selection

Selected appropriate models based on tradeoffs across:

- Latency
- Token usage
- Cost

---

## Infrastructure: Azure AI Foundry

Used **Microsoft Azure AI Foundry** to orchestrate model deployment and routing. This optimization:

- Reduced operational cost by approximately **15%**
- Maintained response quality and system reliability throughout

---

## Summary

This work required balancing **system design**, **API integration**, and **AI behavior tuning** in a production environment — with a strong focus on cost efficiency, reliability, and maintainability.
