# 01. Planning & Design — Feature Overview

## Purpose

Define the problem, intended users, scope and boundaries for AiraMeet Speech-to-Text (STT). This is a planning statement, not evidence of a deployed capability.

## Scope

In scope: transcription of meeting speech in Egyptian Arabic and English, including Arabic-English code-switching; live partial and finalized text; timestamps; association with authenticated meeting participants; persistence and retrieval of finalized transcript segments.

Out of scope for the initial design: acoustic speaker diarization, voice biometrics, translation, meeting summarization, and automatic decisions/actions. These may consume transcripts or be considered separately.

## Prerequisites

Product agreement on supported meeting flows, lawful audio/transcript processing, participant notice/consent, and access to a representative labeled corpus. These are not established in the available project material and remain open.

## Technical approach

Treat the feature as a pipeline of distinct responsibilities: browser audio capture; authenticated backend admission and routing; AI speech activity handling and recognition; backend attribution, event delivery and persistence; client rendering. The backend remains authoritative for meeting membership and identity. The model must be evaluated on representative Egyptian Arabic, English and code-switched material before selection.

## User outcomes

- Participants can follow live captions while speaking in supported languages.
- Reviewers can retrieve finalized meeting text with time references and participant identity.
- Downstream AiraMeet features can consume stable finalized transcript records through a documented interface.

## Deliverables

Requirements, architecture and interfaces; curated dataset manifests; evaluated model artifact; integrated real-time path; verification evidence; and operational runbook.

## Acceptance criteria

Every requirement has an owner and verification method; phase outputs meet the gates defined in their owning documents; no model, performance or deployment result is represented as achieved without evidence.

## Dependencies

Authentication and meeting lifecycle, browser microphone permissions, networking, transcript storage, AI inference capacity, privacy policy and suitable corpus.

## Open questions

1. Which dialects beyond Egyptian Arabic and which English varieties are product requirements?
2. What consent, retention, deletion, residency and access rules apply to audio and transcript text?
3. Is live transcription opt-in, host-controlled, or enabled by default?
4. What are the supported browsers, participant limits and offline/degraded behavior?
