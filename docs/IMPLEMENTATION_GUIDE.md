# Implementation Guide: Stability & Reliability Features

This document provides comprehensive guidance for the four newly implemented stability features:

1. Synthetic Monitoring for Critical Workflows
2. Stable Problem-Details Error Taxonomy
3. Ownership Transfer Acceptance and Expiry
4. Deployment Rollback Verification Checks

---

## 1. Synthetic Monitoring for Critical Workflows

### Overview

Synthetic monitoring probes critical user journeys end-to-end, covering normal operations, failure scenarios, retry behavior, and authorization paths.

### Location

- Core service: `src/synthetic-monitoring/synthetic-monitoring.service.ts`
- Workflow definitions: `src/synthetic-monitoring/workflows/critical-workflows.ts`
- Tests: `src/synthetic-monitoring/workflows/critical-workflows.spec.ts`

### Usage

```typescript
import { SyntheticMonitoringService } from './synthetic-monitoring/synthetic-monitoring.service';
import { CriticalWorkflowsService } from './synthetic-monitoring/workflows/critical-workflows';

// Register critical workflows
const workflowService = new CriticalWorkflowsService(dependencies);
const journeys = workflowService.getAllJourneys();

journeys.forEach(journey => {
  monitoringService.register(journey);
});

// Run all workflows
const results = await monitoringService.runAll();

// Check for failures
results.forEach(result => {
  if (!result.passed) {
    logger.error(`Journey ${result.journeyName} FAILED`);
    alertOpsTeam(result);
  }
});
```

### Covered Workflows

1. **User Authentication Flow**
   - Login with valid credentials
   - Invalid credentials rejection (401)
   - Protected endpoint authorization
   - Token validation

2. **Portfolio Management Flow**
   - List portfolios
   - Create portfolio
   - Unauthorized access denial (403)
   - Not found handling (404)

3. **Payment Settlement Flow**
   - Invoice creation
   - Insufficient funds validation
   - Malformed wallet address rejection
   - Settlement confirmation

4. **Ownership Transfer Flow**
   - Transfer initiation
   - Status retrieval
   - Authorization verification
   - Cancellation handling

### Edge Cases Handled

- **Network timeouts**: Trigger retry with exponential backoff
- **Authorization failures**: Fail fast without retry
- **Rate limiting**: Detected and reported
- **Database connection failures**: Marked as transient, retryable

### Recovery Behavior

All workflows are **read-only probes** by design:
- No mutations occur during synthetic tests
- Failed journeys trigger alerts without affecting production state
- No rollback needed
- Comprehensive step-by-step diagnostics provided

### Scheduling

Run synthetic monitoring on a schedule:

```typescript
import { Cron } from '@nestjs/schedule';

@Injectable()
export class SyntheticMonitoringScheduler {
  @Cron('*/5 * * * *') // Every 5 minutes
  async runSyntheticChecks() {
    const results = await this.monitoringService.runAll();
    await this.publishMetrics(results);
  }
}
```

### Alerting

Integrate with your alerting system:

```typescript
function alertOpsTeam(result: JourneyResult) {
  const failedSteps = result.steps.filter(s => s.status === 'fail');
  
  sendAlert({
    severity: 'critical',
    title: `Synthetic monitoring: ${result.journeyName} failed`,
    description: `${failedSteps.length} step(s) failed`,
    steps: failedSteps.map(s => ({
      name: s.name,
      error: s.error,
      duration: s.durationMs
    }))
  });
}
```

---

## 2. Stable Problem-Details Error Taxonomy

### Overview

RFC 7807-compliant error responses with stable error codes, retry guidance, and recovery instructions.

### Location

- Filter: `src/common/filters/problem-details.filter.ts`
- Taxonomy: `src/common/errors/error-taxonomy.ts`
- Error codes: `src/common/errors/error-codes.ts`
- Tests: `src/common/filters/problem-details.filter.spec.ts`

### Registration

Apply the filter globally in your main application module:

```typescript
import { APP_FILTER } from '@nestjs/core';
import { ProblemDetailsFilter } from './common/filters/problem-details.filter';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: ProblemDetailsFilter,
    },
  ],
})
export class AppModule {}
```

### Response Format

All errors return structured problem-details responses:

```json
{
  "type": "https://trellis-stellar.github.io/Trellis-API/problems/VALIDATION_ERROR",
  "title": "Request validation failed",
  "status": 400,
  "detail": "resourceType and resourceId are required",
  "instance": "/api/ownership/transfers",
  "remediation": "Review the request body and address validation constraints in the 'errors' object.",
  "errorCode": "VALIDATION_ERROR",
  "domain": "validation",
  "retryable": false,
  "correlationId": "req_1234567890_abc123",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Error Domains

- `auth`: Authentication and authorization errors
- `validation`: Request validation failures
- `settlement`: Stellar payment settlement errors
- `payment`: Payment processing errors
- `reconciliation`: Transaction reconciliation issues
- `blockchain`: Blockchain/Horizon communication errors
- `portfolio`: Portfolio management errors
- `rate_limit`: Rate limiting
- `quota`: Quota exceeded
- `system`: Generic system errors

### Retry Behavior

The filter automatically classifies errors as retryable or non-retryable:

**Retryable errors** (include `retryable: true` and `retryAfterSeconds`):
- `DATABASE_ERROR`
- `SERVICE_UNAVAILABLE`
- `DEPENDENCY_TIMEOUT`
- `SETTLEMENT_TIMEOUT`
- `RATE_LIMITED`

**Non-retryable errors**:
- `VALIDATION_ERROR`
- `UNAUTHORIZED`
- `FORBIDDEN`
- `NOT_FOUND`
- `DUPLICATE_ENTRY`

### Custom Exception Usage

Throw custom exceptions with problem-details context:

```typescript
import { ErrorCode } from './common/errors/error-codes';

class OwnershipTransferException extends Error {
  constructor(
    message: string,
    public errorCode: ErrorCode,
    public statusCode: number,
    public details?: unknown,
  ) {
    super(message);
  }
}

throw new OwnershipTransferException(
  'Transfer has expired',
  ErrorCode.PRECONDITION_FAILED,
  412,
  { transferId: 'ot-123', expiresAt: '2024-01-15T10:00:00Z' }
);
```

### Edge Cases Handled

1. **Database errors**: Mapped to specific error codes (unique constraint → `DUPLICATE_ENTRY`)
2. **Timeout errors**: Detected from message content and marked retryable
3. **Authorization errors**: Detected from message patterns
4. **Production sanitization**: Internal error details hidden in production
5. **Correlation ID**: Generated if not provided in request headers

### Recovery Guidance

Every error includes:
- `remediation`: Human-readable recovery steps
- `retryStrategy`: `immediate`, `exponential_backoff`, or `none`
- `retryAfterSeconds`: Suggested retry delay for retryable errors

---

## 3. Ownership Transfer Acceptance and Expiry

### Overview

Two-step ownership transfer workflow with expiry enforcement, audit trail, and idempotency.

### Location

- Service: `src/core/ownership/ownership-transfer.service.ts`
- Entities: `src/core/ownership/entities/ownership-transfer.entity.ts`
- Tests: `src/core/ownership/ownership-transfer.service.spec.ts`

### Workflow States

1. `PENDING`: Transfer initiated, awaiting acceptance
2. `ACCEPTED`: Proposed owner accepted transfer
3. `CANCELLED`: Current owner cancelled transfer
4. `REJECTED`: Proposed owner rejected transfer
5. `EXPIRED`: Transfer expired before action

### API Usage

#### 1. Initiate Transfer

```typescript
const transfer = await ownershipTransferService.initiateTransfer({
  resourceType: 'portfolio',
  resourceId: 'port-123',
  currentOwnerId: 'user-1',
  proposedOwnerId: 'user-2',
  expiresInHours: 72,
  idempotencyKey: 'transfer-request-abc123', // Optional
});
```

#### 2. Accept Transfer (by proposed owner)

```typescript
const accepted = await ownershipTransferService.acceptTransfer(
  transferId,
  'user-2' // Must be proposedOwnerId
);
```

#### 3. Cancel Transfer (by current owner)

```typescript
const cancelled = await ownershipTransferService.cancelTransfer(
  transferId,
  'user-1', // Must be currentOwnerId
  'Changed mind'
);
```

#### 4. Reject Transfer (by proposed owner)

```typescript
const rejected = await ownershipTransferService.rejectTransfer(
  transferId,
  'user-2', // Must be proposedOwnerId
  'Not interested'
);
```

#### 5. Retrieve Transfer

```typescript
const transfer = await ownershipTransferService.getTransfer(
  transferId,
  'user-1' // Optional: enforces participant authorization
);
```

#### 6. Audit History

```typescript
const history = await ownershipTransferService.getAuditHistory(
  transferId,
  'user-1' // Optional: enforces participant authorization
);
```

### Authorization Matrix

| Action   | Current Owner | Proposed Owner | Non-participant |
|----------|---------------|----------------|-----------------|
| Initiate | ✓             | ✗              | ✗               |
| Accept   | ✗             | ✓              | ✗               |
| Cancel   | ✓             | ✗              | ✗               |
| Reject   | ✗             | ✓              | ✗               |
| Read     | ✓             | ✓              | ✗               |

### Edge Cases

#### 1. Expiry Handling

Transfers expire automatically:

```typescript
// Lazy expiry on read
const transfer = await service.getTransfer(transferId);
// If expired, status changes to EXPIRED

// Batch expiry via scheduled job
@Cron('0 * * * *') // Every hour
async expireTransfers() {
  const count = await this.ownershipTransferService.expirePendingTransfers();
  this.logger.log(`Expired ${count} transfers`);
}
```

#### 2. Idempotency

Prevent duplicate transfers with idempotency keys:

```typescript
// First call creates transfer
const t1 = await service.initiateTransfer({
  resourceType: 'portfolio',
  resourceId: 'port-123',
  currentOwnerId: 'user-1',
  proposedOwnerId: 'user-2',
  idempotencyKey: 'key-1',
});

// Retry with same key returns existing transfer
const t2 = await service.initiateTransfer({
  resourceType: 'portfolio',
  resourceId: 'port-123',
  currentOwnerId: 'user-1',
  proposedOwnerId: 'user-2',
  idempotencyKey: 'key-1',
});

expect(t1.id).toBe(t2.id);
```

#### 3. State Transition Guards

```typescript
// Cannot accept cancelled transfer
await expect(
  service.acceptTransfer(cancelledTransferId, 'user-2')
).rejects.toThrow(BadRequestException);

// Cannot cancel accepted transfer
await expect(
  service.cancelTransfer(acceptedTransferId, 'user-1')
).rejects.toThrow(BadRequestException);
```

### Recovery Behavior

1. **Audit Trail**: Every action is logged with actor, timestamp, and reason
2. **Authorization Violations**: Logged as `ACCESS_DENIED` audit events
3. **Failed Transitions**: Preserve original state, throw descriptive error
4. **Expiry**: Automatic transition to `EXPIRED`, audit event recorded

### Database Migration

Generate migration for ownership transfer tables:

```bash
npm run migration:generate -- -n CreateOwnershipTransferTables
npm run migration:run
```

---

## 4. Deployment Rollback Verification Checks

### Overview

Automated rollback safety verification before resuming production traffic after a deployment rollback.

### Location

- Core logic: `src/monitoring/deployment-rollback/deployment-rollback.ts`
- Integration: `src/monitoring/deployment-rollback/deployment-rollback-integration.ts`
- Tests: `src/monitoring/deployment-rollback/*.spec.ts`

### Check Types

1. **version**: Verify deployed version matches rollback target
2. **health**: Ensure service is healthy
3. **schema**: Database schema compatibility
4. **dependency**: External service connectivity (Horizon, Redis, etc.)
5. **authorization**: Verify auth guards fail closed (401/403)

### Usage

#### Basic Verification

```typescript
import { DeploymentRollbackIntegrationService } from './monitoring/deployment-rollback/deployment-rollback-integration';

const target = service.buildStandardProductionTarget(
  'deploy-2024-01-15-01',
  '2.1.0', // from version (failed)
  '2.0.5'  // to version (rollback target)
);

const actor = {
  id: 'ops-user-1',
  roles: ['RELEASE_MANAGER'],
};

const report = await service.verifyRollback(target, envConfig, actor);

if (report.canResumeTraffic) {
  console.log('✓ Safe to resume traffic');
} else {
  console.error('✗ Do NOT resume traffic');
  report.remediation.forEach(r => console.error(r));
}
```

#### Custom Rollback Target

```typescript
const customTarget: RollbackTarget = {
  deploymentId: 'deploy-2024-01-15-01',
  fromVersion: '2.1.0',
  toVersion: '2.0.5',
  environment: 'production',
  checks: [
    { name: 'api-version', kind: 'version' },
    { name: 'health-check', kind: 'health' },
    { name: 'db-schema-version', kind: 'schema' },
    { name: 'stellar-horizon', kind: 'dependency', required: true },
    { name: 'cache-warmup', kind: 'dependency', required: false }, // Warning only
    { name: 'admin-endpoint', kind: 'authorization', requireAuthFailClosed: true },
  ],
};
```

### Verification Report

```typescript
interface RollbackVerificationReport {
  deploymentId: string;
  fromVersion: string;
  toVersion: string;
  environment: string;
  allPassed: boolean;
  canResumeTraffic: boolean;
  passed: number;
  failed: number;
  warnings: number;
  results: RollbackCheckResult[];
  remediation: string[];
  verifiedAt: string;
}
```

### Edge Cases

#### 1. Version Mismatch (Rollback Not Applied)

```
✗ [api-version] Deployment is still running failed version '2.1.0'. 
  Re-apply rollback artifact '2.0.5' before resuming traffic.
```

#### 2. Authorization Bypass Detected

```
✗ [admin-endpoint] Endpoint failed to reject unauthenticated request. 
  Verify auth guards/middleware before resuming traffic.
```

#### 3. Transient Network Failures

Automatic retry with exponential backoff (100ms, 200ms, 400ms):

```typescript
// First attempt: ETIMEDOUT
// Second attempt: Success
// Result: PASSED
```

#### 4. Optional Check Failures

```
⚠ [cache-warmup] Check failed after 3 attempt(s): Cache cold.
  canResumeTraffic: true (warning only, not blocking)
```

### Authorization

Only users with `RELEASE_MANAGER` or `ADMIN` roles can execute rollback verification:

```typescript
const actor = {
  id: 'dev-user-1',
  roles: ['DEVELOPER'],
};

await expect(
  service.verifyRollback(target, envConfig, actor)
).rejects.toThrow(UnauthorizedRollbackVerificationError);
```

### Recovery Behavior

1. **Failed Verification**: Blocks traffic resumption, provides remediation steps
2. **Retry Exhaustion**: After 3 attempts, marks check as failed
3. **Authorization Probe**: Verifies endpoints fail closed (401/403) when accessed without credentials
4. **Audit Logging**: All verification attempts logged with actor, timestamp, and results

### Integration with CI/CD

```yaml
# .github/workflows/rollback.yml
- name: Verify Rollback
  run: |
    npm run rollback:verify -- \
      --deployment-id ${{ github.run_id }} \
      --from-version ${{ env.FAILED_VERSION }} \
      --to-version ${{ env.ROLLBACK_VERSION }} \
      --environment production \
      --actor ${{ github.actor }}
    
- name: Resume Traffic
  if: success()
  run: npm run traffic:resume
```

### Monitoring & Alerting

```typescript
const report = await service.verifyRollback(target, envConfig, actor);

await sendMetrics({
  deployment_id: report.deploymentId,
  rollback_verification_passed: report.allPassed,
  checks_passed: report.passed,
  checks_failed: report.failed,
  checks_warned: report.warnings,
  verified_at: report.verifiedAt,
});

if (!report.canResumeTraffic) {
  await sendAlert({
    severity: 'critical',
    title: `Rollback verification failed: ${report.deploymentId}`,
    description: `${report.failed} check(s) failed. Traffic NOT resumed.`,
    remediation: report.remediation,
  });
}
```

---

## Testing

### Run All Tests

```bash
# Unit tests
npm test

# Coverage report
npm run test:cov

# Specific test suites
npm test -- synthetic-monitoring
npm test -- problem-details
npm test -- ownership-transfer
npm test -- deployment-rollback
```

### Integration Tests

```bash
# E2E tests
npm run test:e2e

# Chaos testing
npm run test:chaos
```

---

## Monitoring & Observability

### Metrics

Expose metrics for each feature:

```typescript
import { Counter, Histogram } from 'prom-client';

// Synthetic monitoring
const syntheticJourneyDuration = new Histogram({
  name: 'synthetic_journey_duration_ms',
  help: 'Duration of synthetic journey execution',
  labelNames: ['journey_name', 'passed'],
});

// Ownership transfers
const ownershipTransferCounter = new Counter({
  name: 'ownership_transfers_total',
  help: 'Total ownership transfers',
  labelNames: ['status', 'resource_type'],
});

// Rollback verifications
const rollbackVerificationCounter = new Counter({
  name: 'rollback_verifications_total',
  help: 'Total rollback verifications',
  labelNames: ['environment', 'passed'],
});
```

### Logging

All features include structured logging with correlation IDs:

```json
{
  "level": "error",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "context": "OwnershipTransferService",
  "message": "FORBIDDEN [req_123_abc] POST /ownership/transfers/ot-123/accept: Unauthorized accept attempt by non-proposed owner",
  "correlationId": "req_123_abc",
  "transferId": "ot-123",
  "actorId": "user-3",
  "expectedActor": "user-2"
}
```

---

## Migration Plan

### Existing Contracts

All features preserve existing contracts:

1. **Synthetic Monitoring**: New feature, no breaking changes
2. **Problem-Details Filter**: Replaces default exception handling, maintains HTTP status codes
3. **Ownership Transfer**: New feature, requires database migration
4. **Rollback Verification**: New feature, optional integration

### Rollout Strategy

1. **Deploy to Staging**: Validate all features in non-production environment
2. **Enable Synthetic Monitoring**: Start collecting baseline metrics
3. **Enable Problem-Details Filter**: Monitor for client compatibility issues
4. **Deploy Ownership Transfer**: Run database migration, enable API endpoints
5. **Integrate Rollback Verification**: Add to deployment pipeline

### Rollback Plan

If issues arise:

1. **Synthetic Monitoring**: Disable scheduled jobs
2. **Problem-Details Filter**: Remove from `APP_FILTER` providers
3. **Ownership Transfer**: Revert database migration
4. **Rollback Verification**: Remove from CI/CD pipeline

---

## Troubleshooting

### Synthetic Monitoring

**Issue**: Journey fails with timeout
- **Cause**: Network latency or slow endpoint
- **Solution**: Increase step timeout or investigate endpoint performance

**Issue**: False positive auth failures
- **Cause**: Test credentials expired
- **Solution**: Rotate test user credentials

### Problem-Details Filter

**Issue**: Clients expect old error format
- **Cause**: Breaking change in error response structure
- **Solution**: Add compatibility layer or update clients

**Issue**: Sensitive data in error responses
- **Cause**: Production sanitization not working
- **Solution**: Verify `NODE_ENV=production`

### Ownership Transfer

**Issue**: Transfers not expiring
- **Cause**: Scheduled expiry job not running
- **Solution**: Verify cron job configuration

**Issue**: Idempotency conflicts
- **Cause**: Same key used for different payloads
- **Solution**: Generate unique keys per request

### Rollback Verification

**Issue**: Version check fails despite correct version
- **Cause**: Load balancer caching stale responses
- **Solution**: Add cache-busting headers or wait for cache TTL

**Issue**: Authorization check passes incorrectly
- **Cause**: Auth middleware not enforcing fail-closed
- **Solution**: Fix auth guards before resuming traffic

---

## Support

For questions or issues:

1. Review this implementation guide
2. Check test files for usage examples
3. Review inline code documentation
4. Contact the platform team

---

*Last updated: 2024-01-15*
