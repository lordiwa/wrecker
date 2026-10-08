---
# Header: YAML between the two "---" lines. Lines starting with # are comments.
id: UC-LOGIN
name: Sign in
actor: User                  # who performs the main flow
level: user_goal
scope: web_app
spec_version: login-v1       # change it whenever the spec changes

# Optional: which attacker roles may see each field (1 = everyone, 4 = only the adversary).
fog:
  injects: 4
  success_postcondition: 4
  failure_postcondition: 4

preconditions:
  - the user has an account
trigger: "The user opens the sign-in page"

# The world a play starts from. Conditions and effects read and write these paths.
initial_world:
  signed_in: false
  failed_attempts: 0
  account_locked: false
  screen: home

# What must be true when a play reaches end_success / end_failure.
success_postcondition:
  all:
    - { op: eq, path: signed_in, value: true }
    - { op: eq, path: account_locked, value: false }
failure_postcondition:
  all:
    - { op: eq, path: signed_in, value: false }
    - { op: eq, path: account_locked, value: true }

# Every world path a "requires" or "applies_when" reads.
referenced_states:
  - account_locked
  - failed_attempts

# Faults Wrecker may inject in the middle of a play.
injects:
  - name: locked_by_admin
    family: data
    applies_when: { op: eq, path: account_locked, value: false }
    effect: { account_locked: true }
---

## Main flow

1. [open_sign_in] User: opens the sign-in page → the system shows the form.
   - effect: { screen: sign_in_form }
   - goto: 2
2. [submit_right_password] User: submits the right email and password → the system signs the user in.
   - requires: { op: eq, path: account_locked, value: false }
   - effect: { signed_in: true, failed_attempts: 0, screen: dashboard }
   - goto: end_success

## Alternate flows

- 2a. condition: the password is wrong (fewer than 4 failures so far)
  - anchor: 2
  - applies_when: { any: [ { op: eq, path: failed_attempts, value: 0 }, { op: eq, path: failed_attempts, value: 1 }, { op: eq, path: failed_attempts, value: 2 }, { op: eq, path: failed_attempts, value: 3 } ] }
  - 2a1. [submit_wrong_password] User: submits a wrong password → the system shows "Wrong email or password".
    - effect: { failed_attempts: { inc: 1 }, screen: sign_in_error }
    - goto: 2

## Exception flows

- 2b. condition: the fifth wrong password in a row
  - anchor: 2
  - applies_when: { op: eq, path: failed_attempts, value: 4 }
  - recovery: the user resets the password from the email link
  - 2b1. [submit_fifth_wrong_password] User: submits a wrong password again → the system locks the account and says so.
    - effect: { failed_attempts: { inc: 1 }, account_locked: true, screen: account_locked }
    - goto: end_failure
