---
id: what-is-a-retry-policy
title: What is a Retry Policy?
sidebar_label: Retry Policy
description: A Retry Policy is collection of attributes that instructs the Temporal Server how to retry a failure of a Workflow Execution or an Activity Execution.
tags:
  - explanation
---

A Retry Policy is collection of attributes that instructs the Temporal Server how to retry a failure of a [Workflow Execution](/docs/concepts/what-is-a-workflow-execution) or an [Activity Task Execution](/docs/concepts/what-is-an-activity-task-execution).

- If a custom Retry Policy is to be used, it must be provided as an options parameter when a Workflow Execution or an Activity Execution is spawned.

- The wait time before a retry is the _retry interval_.
  A retry interval is the smaller of two values:
  - The [Initial Interval](#initial-interval) multiplied by the [Backoff Coefficient](#backoff-coefficient) raised to the power of the number of retries.
  - The [Maximum Interval](#maximum-interval).

<!-- ![Diagram that shows the retry interval and its formula](/img/retry-interval-diagram.png) -->

- When a Workflow Execution is spawned it is not associated with a default Retry Policy and thus does not retry by default.
  The intention is that a Workflow Definition should be written to never fail due to intermittent issues; an Activity is designed to handle such issues.

Retry Policies do not apply to [Workflow Task Executions](/docs/concepts/what-is-a-workflow-task-execution), which, by default, retry indefinitely.

- A Retry Policy can be provided to a Workflow Execution when it is spawned, but only certain scenarios merit doing this, such as the following:

  - A [Temporal Cron Job](/docs/concepts/what-is-a-temporal-cron-job) or some other stateless, always-running Workflow Execution that can benefit from retries.
  - A file-processing or media-encoding Workflow Execution that downloads files to a host.

- When an [Activity Execution](/docs/concepts/what-is-an-activity-execution) is spawned, it is associated with a default Retry Policy, and thus Activity Task Executions are retried by default.
  When an Activity Task Execution is retried, the Cluster places a new [Activity Task](/docs/concepts/what-is-an-activity-task) into its respective [Activity Task Queue](/docs/concepts/what-is-a-task-queue), which results in a new Activity Task Execution.

There are some subtle nuances to how Events are recorded to an Event History when a Retry Policy comes into play.

- For an Activity Execution, the [ActivityTaskStarted](/docs/concepts/what-is-an-event#activitytaskstarted) Event will not show up in the Workflow Execution Event History until the Activity Execution has completed or failed (having exhausted all retries).
  This is to avoid filling the Event History with noise.
  Use the Describe API to get a pending Activity Execution's attempt count.

- For a Workflow Execution with a Retry Policy, if the Workflow Execution fails, the Workflow Execution will [Continue-As-New](/docs/concepts/what-is-continue-as-new) and the associated Event is written to the Event History.
  The [WorkflowExecutionContinuedAsNew](/docs/concepts/what-is-an-event#workflowexecutioncontinuedasnew) Event will have an "initiator" field that will specify the Retry Policy as the value and the new Run Id for the next retry attempt.
  The new Workflow Execution is created immediately.
  But the first Workflow Task won't be scheduled until the backoff duration is exhausted.
  That duration is recorded as the `firstWorkflowTaskBackoff` field of the new run's `WorkflowExecutionStartedEventAttributes` event.

### Default values for Retry Policy\*\*

```
Initial Interval     = 1 second
Backoff Coefficient  = 2.0
Maximum Interval     = 100 × Initial Interval
Maximum Attempts     = ∞
Non-Retryable Errors = []
```

### Initial Interval

- **Description**: Amount of time that must elapse before the first retry occurs.
  - **The default value is 1 second.**
- **Use case**: This is used as the base interval time for the [Backoff Coefficient](#backoff-coefficient) to multiply against.

### Backoff Coefficient

- **Description**: The value dictates how much the _retry interval_ increases.
  - **The default value is 2.0.**
  - A backoff coefficient of 1.0 means that the retry interval always equals the [Initial Interval](#initial-interval).
- **Use case**: Use this attribute to increase the interval between retries.
  By having a backoff coefficient greater than 1.0, the first few retries happen relatively quickly to overcome intermittent failures, but subsequent retries happen farther and farther apart to account for longer outages.
  Use the [Maximum Interval](#maximum-interval) attribute to prevent the coefficient from increasing the retry interval too much.

### Maximum Interval

- **Description**: Specifies the maximum interval between retries.
  - **The default value is 100 times the [Initial Interval](#initial-interval).**
- **Use case**: This attribute is useful for [Backoff Coefficients](#backoff-coefficient) that are greater than 1.0 because it prevents the retry interval from growing infinitely.

### Maximum Attempts

- **Description**: Specifies the maximum number of execution attempts that can be made in the presence of failures.
  - **The default is unlimited.**
  - If this limit is exceeded, the execution fails without retrying again. When this happens an error is returned.
  - Setting the value to 0 also means unlimited.
  - Setting the value to 1 means a single execution attempt and no retries.
  - Setting the value to a negative integer results in an error when the execution is invoked.
- **Use case**: Use this attribute to ensure that retries do not continue indefinitely.
  However, in the majority of cases, we recommend relying on the Workflow Execution Timeout, in the case of [Workflows](#workflow), or Schedule-To-Close Timeout, in the case of [Activities](#activity), to limit the total duration of retries instead of using this attribute.

### Non-Retryable Errors

- **Description**: Specifies errors that shouldn't be retried.
  - **Default is none.**
  - If one of those errors occurs, the [Activity Task Execution](#activity-task-execution) or [Workflow Execution](#workflow-execution) is not retried.
- **Use case**: There may be errors that you know of that should not trigger a retry.
  In this case you can specify them such that if they occur, the given execution will not be retried.

**Implementation guides:**

- [How to set a Retry Policy for a Workflow Execution in Go](/docs/go/how-to-set-startworkflowoptions-in-go/#retrypolicy)
- [How to set a custom Retry Policy for Activity Task Executions in Go](/docs/go/how-to-set-activityoptions-in-go/#retrypolicy)
