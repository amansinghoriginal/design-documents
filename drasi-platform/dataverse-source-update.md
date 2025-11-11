# Dataverse Source Update

* Project Drasi - 2025-11-11 - Ruokun Niu (@ruokun-niu)

## Overview

The Dataverse Source is a critical component of Drasi that enables integration with Microsoft Dataverse, allowing continuous queries to monitor and react to data changes in Dataverse environments. The current implementation has several limitations that impact its security, performance, and maintainability. This design document outlines the updates needed to modernize the Dataverse Source implementation.

The primary improvements include: adding support for managed identity authentication to enhance security, migrating to the Drasi .NET Source SDK for better maintainability and consistency with other sources, implementing a more efficient polling algorithm to reduce unnecessary API calls, and expanding data type support to handle complex Dataverse types like lookups.

These updates will make the Dataverse Source more secure, efficient, and easier to maintain while expanding its capability to handle a broader range of Dataverse data scenarios.

## Terms and definitions

| Term | Definition |
|------|------------|
| Managed Identity | Azure Active Directory (AAD) authentication mechanism that provides an automatically managed identity for Azure resources to authenticate to services without storing credentials in code |
| Dataverse | Microsoft's cloud-based data platform that provides secure storage and management of business data |
| Lookup | A Dataverse data type that represents a reference to a record in another table, similar to a foreign key relationship |
| Polling Interval | The time period between consecutive checks for new data changes in the source system |
| Reactivator | The Drasi component responsible for detecting and processing data changes from the source |
| Proxy | The Drasi component that provides initial data and handles queries to the source system |
| Source SDK | A software development kit that provides common patterns and utilities for building Drasi sources |

## Objectives

### User scenarios

**Scenario 1: Secure Enterprise Integration**
- **Persona**: Enterprise IT Administrator
- **Goal**: Deploy Drasi with Dataverse Source in a production environment without storing credentials
- **Requirements**: The administrator needs to authenticate using Azure Managed Identity instead of storing service principal credentials in configuration files

**Scenario 2: Efficient Change Detection**
- **Persona**: DevOps Engineer
- **Goal**: Monitor Dataverse data changes with minimal API call overhead
- **Requirements**: The system should intelligently detect when changes occur and adjust polling frequency accordingly, reducing costs and improving responsiveness

**Scenario 3: Complex Data Modeling**
- **Persona**: Business Application Developer
- **Goal**: Create continuous queries that involve related Dataverse entities
- **Requirements**: The developer needs to query and track changes across entities with lookup relationships

### Goals

- **Security Enhancement**: Implement managed identity authentication to eliminate the need to store credentials, improving security posture
- **SDK Migration**: Migrate to the Drasi .NET Source SDK to ensure consistency with other sources and leverage common functionality
- **Performance Optimization**: Implement an adaptive polling algorithm that reduces API calls during quiet periods while maintaining responsiveness during active periods
- **Data Type Support**: Add support for the Dataverse lookup data type to enable queries across related entities
- **Documentation**: Provide comprehensive documentation for configuration, authentication options, and data type handling

### Non-Goals

- Implementing real-time streaming from Dataverse (continue using polling-based approach)
- Supporting all Dataverse data types in the initial release (focus on lookup type specifically)
- Backward compatibility with the current implementation's configuration format
- Supporting authentication methods beyond managed identity and existing service principal authentication
- Performance optimization of the query execution itself (focus is on change detection)

## Design requirements

### Requirements

**Track issue**: https://github.com/drasi-project/drasi-platform/issues/338

**Functional Requirements**:
1. Support authentication via Azure Managed Identity
2. Maintain support for existing service principal authentication as a fallback
3. Use the Drasi .NET Source SDK for both proxy and reactivator components
4. Implement an adaptive polling algorithm that adjusts based on change frequency
5. Handle Dataverse lookup data types correctly in both initial load and change detection
6. Provide clear error messages for authentication and configuration issues

**Non-Functional Requirements**:
1. Maintain or improve current performance for initial data load
2. Reduce API call volume by at least 50% during periods with no changes
3. Ensure backward compatibility of the query interface (users should not need to change their continuous queries)
4. Follow Drasi security best practices for credential handling
5. Provide comprehensive logging and telemetry for troubleshooting

**Extensibility**:
- Design should allow easy addition of new Dataverse data types in the future
- Polling algorithm should be configurable to allow tuning for different environments
- Authentication mechanism should be extensible for future authentication types

### Dependencies

**Drasi .NET Source SDK**:
- The current .NET Source SDK must support all required functionality for Dataverse integration
- If the SDK lacks necessary features (e.g., specific authentication patterns), it must be extended

**Azure Identity Libraries**:
- Requires Azure.Identity NuGet package for managed identity support
- Must be compatible with the .NET version used by the SDK

**Dataverse SDK**:
- Microsoft.PowerPlatform.Dataverse.Client or equivalent for Dataverse API access
- Must support both managed identity and service principal authentication

**Control Plane**:
- Configuration schema changes for authentication options may require control plane updates
- Resource provider should support deploying sources with managed identity configuration

**Build and Release Pipelines**:
- GitHub Actions workflows must be updated to build and test the updated Dataverse source
- Both `build-test` and `draft-release` workflows need updates

### Out of scope

**Real-time Change Feed**: Dataverse does not currently provide a native change feed or push notification mechanism, so this design will continue using polling. Future Dataverse platform updates may enable this.

**Complete Data Type Coverage**: While lookup types are critical, supporting all Dataverse data types (OptionSet, MultiSelectOptionSet, Customer, etc.) is deferred to future iterations.

**Multi-tenant Authentication**: This design focuses on single-tenant scenarios. Multi-tenant authentication patterns are not addressed.

**Historical Data Analysis**: The design focuses on forward-looking change detection, not analyzing historical change patterns beyond the current polling state.

## Design

### High-level design

The updated Dataverse Source will consist of two main components, both refactored to use the Drasi .NET Source SDK:

1. **Dataverse Reactivator**: Responsible for detecting changes in Dataverse and publishing them to the Drasi query container
   - Uses adaptive polling algorithm to check for changes
   - Authenticates using managed identity or service principal
   - Tracks change tokens to detect incremental changes
   - Handles lookup type expansion

2. **Dataverse Proxy**: Provides initial data snapshots and handles node queries
   - Authenticates using the same mechanism as reactivator
   - Translates Drasi queries to Dataverse FetchXML queries
   - Resolves lookup relationships during initial load

Both components will share common authentication and configuration logic through the SDK.

**Key Improvements**:
- **Adaptive Polling**: Instead of a fixed interval, the system will use an exponential backoff when no changes are detected, with a configurable minimum and maximum interval
- **Managed Identity**: Leverages Azure managed identity when running in Azure, eliminating credential storage
- **SDK-based Architecture**: Common patterns for configuration, logging, and telemetry provided by the SDK

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Drasi Query Container                    │
│                                                              │
│  ┌────────────────┐                  ┌──────────────────┐  │
│  │  Initial Load  │◄─────────────────┤ Dataverse Proxy  │  │
│  │   (Snapshot)   │                  │                  │  │
│  └────────────────┘                  └──────────────────┘  │
│         ▲                                      │            │
│         │                                      │            │
│         │                                      ▼            │
│  ┌──────┴───────┐                  ┌──────────────────┐    │
│  │   Change     │◄─────────────────┤   Dataverse      │    │
│  │   Stream     │                  │  Reactivator     │    │
│  └──────────────┘                  └──────────────────┘    │
│                                              │              │
└──────────────────────────────────────────────┼──────────────┘
                                               │
                                               │ Adaptive
                                               │ Polling
                                               ▼
                           ┌──────────────────────────────────┐
                           │   Microsoft Dataverse            │
                           │                                  │
                           │  ┌────────────┐ ┌─────────────┐ │
                           │  │  Account   │ │  Contact    │ │
                           │  │  Table     │ │  Table      │ │
                           │  └────────────┘ └─────────────┘ │
                           │                                  │
                           │  Authentication:                 │
                           │  - Managed Identity              │
                           │  - Service Principal             │
                           └──────────────────────────────────┘

SDK Components (Shared):
┌────────────────────────────────────────────────┐
│  Drasi .NET Source SDK                         │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Config       │  │ Authentication       │   │
│  │ Management   │  │ Provider             │   │
│  └──────────────┘  └──────────────────────┘   │
│  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Logging &    │  │ Change Publisher     │   │
│  │ Telemetry    │  │ Interface            │   │
│  └──────────────┘  └──────────────────────┘   │
└────────────────────────────────────────────────┘
```

### Detail design

#### 1. Authentication Mechanism

**Managed Identity Authentication**:
```csharp
// Using Azure.Identity for managed identity
public interface IDataverseAuthProvider
{
    Task<ServiceClient> GetAuthenticatedClientAsync();
}

public class ManagedIdentityAuthProvider : IDataverseAuthProvider
{
    private readonly string _instanceUrl;
    private readonly ManagedIdentityCredential _credential;
    
    public ManagedIdentityAuthProvider(string instanceUrl, string clientId = null)
    {
        _instanceUrl = instanceUrl;
        _credential = clientId != null 
            ? new ManagedIdentityCredential(clientId)
            : new ManagedIdentityCredential();
    }
    
    public async Task<ServiceClient> GetAuthenticatedClientAsync()
    {
        var token = await _credential.GetTokenAsync(
            new TokenRequestContext(new[] { $"{_instanceUrl}/.default" }));
        
        return new ServiceClient(
            instanceUri: new Uri(_instanceUrl),
            tokenProviderFunction: async (uri) => token.Token);
    }
}

public class ServicePrincipalAuthProvider : IDataverseAuthProvider
{
    // Existing implementation for backward compatibility
}
```

**Configuration Schema**:
```yaml
# Example source configuration
kind: Source
apiVersion: v1
name: my-dataverse
spec:
  kind: Dataverse
  properties:
    instanceUrl: https://myorg.crm.dynamics.com
    
    # Option 1: Managed Identity (preferred)
    authentication:
      type: ManagedIdentity
      # Optional: specify client ID for user-assigned managed identity
      clientId: "optional-client-id"
    
    # Option 2: Service Principal (fallback)
    # authentication:
    #   type: ServicePrincipal
    #   clientId: "app-client-id"
    #   clientSecret: "app-secret"
    #   tenantId: "tenant-id"
    
    # Polling configuration
    polling:
      initialInterval: 5s
      minInterval: 5s
      maxInterval: 300s
      backoffMultiplier: 2.0
    
    # Tables to monitor
    tables:
      - name: account
      - name: contact
```

#### 2. Adaptive Polling Algorithm

**Polling State Machine**:
```csharp
public class AdaptivePollingController
{
    private TimeSpan _currentInterval;
    private readonly TimeSpan _minInterval;
    private readonly TimeSpan _maxInterval;
    private readonly double _backoffMultiplier;
    private DateTime _lastChangeDetected;
    
    public AdaptivePollingController(PollingConfig config)
    {
        _minInterval = config.MinInterval;
        _maxInterval = config.MaxInterval;
        _backoffMultiplier = config.BackoffMultiplier;
        _currentInterval = config.InitialInterval;
    }
    
    public TimeSpan GetNextInterval(bool changesDetected)
    {
        if (changesDetected)
        {
            // Reset to minimum interval on change detection
            _currentInterval = _minInterval;
            _lastChangeDetected = DateTime.UtcNow;
        }
        else
        {
            // Exponential backoff up to max interval
            _currentInterval = TimeSpan.FromSeconds(
                Math.Min(
                    _currentInterval.TotalSeconds * _backoffMultiplier,
                    _maxInterval.TotalSeconds
                )
            );
        }
        
        return _currentInterval;
    }
}
```

**Change Detection Flow**:
1. Query Dataverse for changes since last change token
2. If changes found:
   - Publish changes to query container
   - Update change token
   - Reset polling interval to minimum
3. If no changes:
   - Increase polling interval using backoff multiplier
   - Wait for next interval

#### 3. SDK Integration

**Reactivator Implementation**:
```csharp
public class DataverseReactivator : SourceReactivator
{
    private readonly IDataverseAuthProvider _authProvider;
    private readonly AdaptivePollingController _pollingController;
    private readonly ILogger _logger;
    
    protected override async Task ProcessChangesAsync(CancellationToken ct)
    {
        var client = await _authProvider.GetAuthenticatedClientAsync();
        
        while (!ct.IsCancellationRequested)
        {
            try
            {
                var changes = await DetectChangesAsync(client);
                var hasChanges = changes.Any();
                
                if (hasChanges)
                {
                    await PublishChangesAsync(changes);
                }
                
                var nextInterval = _pollingController.GetNextInterval(hasChanges);
                await Task.Delay(nextInterval, ct);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error detecting changes");
                await Task.Delay(_pollingController.GetNextInterval(false), ct);
            }
        }
    }
}
```

**Proxy Implementation**:
```csharp
public class DataverseProxy : SourceProxy
{
    private readonly IDataverseAuthProvider _authProvider;
    
    protected override async Task<IEnumerable<Node>> GetNodesAsync(
        string nodeType, 
        CancellationToken ct)
    {
        var client = await _authProvider.GetAuthenticatedClientAsync();
        
        // Convert node type to Dataverse table name
        var tableName = MapNodeTypeToTable(nodeType);
        
        // Build and execute FetchXML query
        var fetchXml = BuildFetchXmlQuery(tableName);
        var results = await client.RetrieveMultipleAsync(
            new FetchExpression(fetchXml));
        
        // Convert entities to nodes with lookup resolution
        return await ConvertEntitiesToNodesAsync(results.Entities);
    }
}
```

#### 4. Lookup Type Handling

**Lookup Resolution**:
```csharp
public class LookupResolver
{
    private readonly ServiceClient _client;
    
    public async Task<Node> ResolveLookupsAsync(Entity entity)
    {
        var node = new Node
        {
            Id = entity.Id.ToString(),
            Labels = new[] { entity.LogicalName }
        };
        
        foreach (var attribute in entity.Attributes)
        {
            if (attribute.Value is EntityReference lookup)
            {
                // Store lookup as a relationship edge
                node.Properties[attribute.Key] = new
                {
                    Id = lookup.Id.ToString(),
                    LogicalName = lookup.LogicalName,
                    Name = lookup.Name
                };
            }
            else
            {
                node.Properties[attribute.Key] = attribute.Value;
            }
        }
        
        return node;
    }
}
```

**Change Token Management**:
```csharp
public class ChangeTokenManager
{
    private Dictionary<string, string> _tableTokens = new();
    
    public async Task<EntityCollection> GetChangesAsync(
        ServiceClient client, 
        string tableName)
    {
        var fetchXml = BuildChangeTrackingQuery(
            tableName, 
            _tableTokens.GetValueOrDefault(tableName));
        
        var results = await client.RetrieveMultipleAsync(
            new FetchExpression(fetchXml));
        
        // Update token from response
        if (results.EntityName != null)
        {
            _tableTokens[tableName] = results.PagingCookie;
        }
        
        return results;
    }
    
    private string BuildChangeTrackingQuery(string tableName, string token)
    {
        // Use Dataverse change tracking capabilities
        return $@"
            <fetch version='1.0' output-format='xml-platform' mapping='logical'>
              <entity name='{tableName}'>
                <all-attributes />
                {(token != null ? $"<filter type='and'><condition attribute='modifiedon' operator='gt' value='{token}' /></filter>" : "")}
              </entity>
            </fetch>";
    }
}
```

#### Advantages of this design
- **Security**: Managed identity eliminates credential storage and rotation concerns
- **Performance**: Adaptive polling reduces API calls significantly during quiet periods
- **Maintainability**: SDK integration ensures consistency with other sources and reduces code duplication
- **Extensibility**: Lookup handling pattern can be extended to other complex types
- **Flexibility**: Configuration allows tuning for different environments and requirements

#### Disadvantages
- **Complexity**: Adaptive polling adds state management complexity
- **Azure Dependency**: Managed identity works best in Azure environments (though service principal remains available)
- **Migration**: Existing deployments will need configuration updates
- **Testing**: Adaptive polling behavior requires comprehensive testing across different change patterns

### API Design

**Configuration API Changes**:

The source configuration schema will be extended to support authentication options:

```yaml
# New authentication section
authentication:
  type: ManagedIdentity | ServicePrincipal
  
  # For ManagedIdentity
  clientId?: string  # Optional: user-assigned managed identity
  
  # For ServicePrincipal (existing)
  clientId?: string
  clientSecret?: string
  tenantId?: string

# New polling configuration section
polling:
  initialInterval: duration    # e.g., "5s", "1m"
  minInterval: duration         # Minimum polling interval
  maxInterval: duration         # Maximum polling interval
  backoffMultiplier: number     # Multiplier for backoff (e.g., 2.0)
```

**CLI Impact**:

No CLI command changes are required. The existing `drasi apply` command will support the new configuration schema through the standard source YAML files.

Example:
```bash
drasi apply -f dataverse-source.yaml
```

**SDK API Extensions**:

If the current .NET Source SDK doesn't support custom authentication providers, we may need to extend it:

```csharp
// Potential SDK extension
public abstract class SourceReactivator
{
    // Add support for custom authentication
    protected virtual Task<IAuthenticationProvider> GetAuthenticationProviderAsync()
    {
        // Default implementation
    }
}
```

### Alternatives Considered

#### Alternative 1: Webhook-based Change Notification

**Description**: Use Dataverse webhooks to receive push notifications instead of polling.

**Advantages**:
- Real-time change notification
- No API call overhead for checking changes
- More efficient for high-frequency changes

**Disadvantages**:
- Requires exposing an HTTP endpoint accessible from Dataverse
- Complex networking setup in Kubernetes environments
- Webhook delivery reliability concerns
- Initial load still requires polling pattern
- Not all Dataverse deployments support webhooks

**Justification for rejection**: The complexity and infrastructure requirements outweigh the benefits, especially since adaptive polling significantly reduces API overhead. This could be reconsidered in the future.

#### Alternative 2: Continue Without SDK Migration

**Description**: Update authentication and polling without migrating to SDK.

**Advantages**:
- Less refactoring required
- Faster initial implementation

**Disadvantages**:
- Misses opportunity for consistency with other sources
- Duplicate code for logging, telemetry, configuration
- Harder to maintain long-term
- No benefit from SDK improvements

**Justification for rejection**: The SDK migration is a key requirement that provides long-term maintainability benefits and consistency across sources.

#### Alternative 3: Fixed Interval with Rate Limiting

**Description**: Keep fixed polling interval but implement rate limiting to reduce API calls.

**Advantages**:
- Simpler implementation
- Predictable behavior

**Disadvantages**:
- Still makes unnecessary API calls during quiet periods
- Doesn't reduce API costs as effectively
- Less responsive during active periods

**Justification for rejection**: Adaptive polling provides better balance between responsiveness and efficiency.

## Security

### Threat Analysis

**Threat 1: Credential Exposure**
- **Risk**: Service principal credentials stored in configuration could be exposed
- **Mitigation**: 
  - Prefer managed identity authentication (no stored credentials)
  - When service principal is required, store credentials in Kubernetes secrets
  - Support Azure Key Vault integration for credential storage
  - Log warnings when using service principal authentication

**Threat 2: Unauthorized Data Access**
- **Risk**: Overly permissive authentication could allow access to unintended data
- **Mitigation**:
  - Document principle of least privilege for managed identity permissions
  - Managed identity should only have read permissions to specified tables
  - Implement table-level filtering in configuration
  - Log all authentication attempts and data access

**Threat 3: Token Theft**
- **Risk**: Access tokens could be intercepted or stolen
- **Mitigation**:
  - Use Azure.Identity library which handles token caching and renewal securely
  - Tokens are short-lived and automatically refreshed
  - Communication with Dataverse uses HTTPS only
  - No logging of tokens or sensitive authentication data

**Threat 4: Man-in-the-Middle Attacks**
- **Risk**: Network traffic could be intercepted
- **Mitigation**:
  - All communication with Dataverse uses HTTPS/TLS
  - Certificate validation enabled by default
  - No option to disable certificate validation

### Security Best Practices

1. **Default to Managed Identity**: Documentation should recommend managed identity as the primary authentication method
2. **Audit Logging**: All authentication events and configuration changes should be logged
3. **Minimal Permissions**: Sample configurations should demonstrate minimal required permissions
4. **Secret Management**: Clear documentation on using Kubernetes secrets or Azure Key Vault for any stored credentials

## Compatibility impact

### Breaking Changes

1. **Configuration Schema**: The authentication configuration format is new and incompatible with the existing format
   - **Migration Path**: Provide documentation and examples for converting old configuration to new format
   - **Tool Support**: Consider providing a configuration migration script

2. **Lookup Data Format**: Lookup values will be represented differently in node properties
   - **Impact**: Existing continuous queries may need updates if they reference lookup fields
   - **Mitigation**: Document the new format and provide query migration examples

### Backward Compatibility

**Preserved**:
- Query interface remains the same (users don't need to change continuous queries except for lookup handling)
- Deployment model unchanged (still deployed as standard Drasi source)
- Existing service principal authentication supported as fallback

**Not Preserved**:
- Configuration file format requires updates
- Internal change detection mechanism is completely different

**Migration Strategy**:
1. Document configuration differences
2. Provide side-by-side configuration examples
3. Consider supporting a "legacy" mode temporarily if needed
4. Update all tutorial and sample configurations

## Supportability

### Telemetry

**Metrics to Collect**:
- `dataverse_polling_interval_seconds`: Current polling interval (gauge)
- `dataverse_changes_detected_total`: Count of polling cycles with changes (counter)
- `dataverse_api_calls_total`: Total API calls to Dataverse (counter)
- `dataverse_authentication_failures_total`: Authentication failure count (counter)
- `dataverse_lookup_resolutions_total`: Lookup resolution count (counter)
- `dataverse_entities_processed_total`: Total entities processed (counter)

**Logs to Generate**:
- Authentication method used and success/failure
- Polling interval adjustments and reasons
- Change detection results (summary, not full data)
- Lookup resolution operations
- Configuration validation errors
- API errors and retry attempts

**Traces**:
- End-to-end request tracing for change detection
- Authentication flow tracing
- Lookup resolution tracing

### Verification

**Unit Tests**:
- Authentication provider tests (managed identity and service principal)
- Adaptive polling algorithm tests with various change patterns
- Lookup resolution logic tests
- Configuration parsing and validation tests

**Integration Tests**:
- End-to-end test with test Dataverse instance
- Authentication with managed identity (in Azure environment)
- Authentication with service principal
- Change detection across different intervals
- Lookup relationship handling
- Error handling and recovery

**E2E Tests**:
- Deploy source with managed identity in Azure Kubernetes Service
- Deploy source with service principal in any Kubernetes cluster
- Verify initial data load with lookups
- Verify change detection and publication
- Verify adaptive polling behavior over time
- Test configuration updates and source restart

**Performance Tests**:
- Measure API call reduction with adaptive polling
- Initial load performance with large datasets
- Change processing latency
- Resource consumption (CPU, memory)

## Development Plan

### Phase 1: SDK Migration (2 weeks)
- Refactor reactivator to use .NET Source SDK
- Refactor proxy to use .NET Source SDK
- Migrate existing authentication to SDK patterns
- Update unit tests
- **Deliverable**: Working source using SDK with existing features

### Phase 2: Managed Identity Authentication (1 week)
- Implement ManagedIdentityAuthProvider
- Add configuration schema for authentication options
- Update documentation
- Add authentication tests
- **Deliverable**: Source supporting both managed identity and service principal

### Phase 3: Adaptive Polling (1 week)
- Implement AdaptivePollingController
- Add polling configuration schema
- Add telemetry for polling behavior
- Add polling algorithm tests
- **Deliverable**: Source with adaptive polling enabled

### Phase 4: Lookup Type Support (1 week)
- Implement LookupResolver
- Update initial load to resolve lookups
- Update change detection to handle lookup changes
- Add lookup handling tests
- **Deliverable**: Source with full lookup support

### Phase 5: Integration and Testing (1 week)
- Integration testing with real Dataverse instance
- E2E testing in Azure and non-Azure environments
- Performance testing and tuning
- Documentation completion
- **Deliverable**: Production-ready source

### Phase 6: Build Pipeline Updates (0.5 weeks)
- Update `build-test` workflow
- Update `draft-release` workflow
- Test automated builds and releases
- **Deliverable**: Automated CI/CD for updated source

**Total Estimated Effort**: 6.5 weeks

### Work Items
1. SDK migration for reactivator
2. SDK migration for proxy
3. Managed identity authentication implementation
4. Service principal fallback implementation
5. Adaptive polling algorithm implementation
6. Lookup type resolver implementation
7. Configuration schema updates
8. Unit test suite
9. Integration test suite
10. E2E test suite
11. Documentation updates
12. Build pipeline updates
13. Sample configurations and tutorials

## Open issues

**Q1: Should we support user-assigned managed identities in addition to system-assigned?**
- **Discussion**: User-assigned identities provide more flexibility but add configuration complexity. Should we support both or start with system-assigned only?
- **Recommendation**: Support both, as user-assigned identities are common in enterprise scenarios

**Q2: What should the default polling intervals be?**
- **Discussion**: Need to balance responsiveness vs API costs. Different environments may have different needs.
- **Recommendation**: Start with conservative defaults (min: 5s, max: 5m, initial: 30s) and allow configuration

**Q3: How should we handle lookup chains (lookup to lookup)?**
- **Discussion**: Should we resolve nested lookups automatically, require explicit configuration, or limit to one level?
- **Recommendation**: Start with single-level lookups, add configuration for depth in future if needed

**Q4: Should we implement any caching for lookup resolutions?**
- **Discussion**: Caching could reduce API calls but adds complexity and staleness concerns.
- **Recommendation**: No caching in initial implementation; revisit based on performance metrics

**Q5: How should we handle Dataverse throttling?**
- **Discussion**: Dataverse has API rate limits. Should we implement backoff or rely on SDK built-in handling?
- **Recommendation**: Implement exponential backoff for throttling responses, coordinated with adaptive polling

**Q6: Should the SDK be extended or should we work around limitations?**
- **Discussion**: If SDK lacks features, should we extend it or implement workarounds in the source?
- **Recommendation**: Extend SDK when features are generally useful, workaround for Dataverse-specific needs

## Appendices

### Appendix A: Dataverse Change Tracking Overview

Dataverse provides change tracking capabilities that maintain a history of data changes. The change tracking feature:
- Tracks create, update, and delete operations
- Provides a change token (paging cookie) to query incremental changes
- Retains changes for a configurable retention period (default 90 days)
- Works with FetchXML or Web API queries

### Appendix B: Sample Configuration Migration

**Old Configuration**:
```yaml
kind: Source
name: my-dataverse
spec:
  kind: Dataverse
  properties:
    url: https://myorg.crm.dynamics.com
    clientId: xxxxx
    clientSecret: xxxxx
    tenantId: xxxxx
```

**New Configuration**:
```yaml
kind: Source
name: my-dataverse
spec:
  kind: Dataverse
  properties:
    instanceUrl: https://myorg.crm.dynamics.com
    authentication:
      type: ServicePrincipal
      clientId: xxxxx
      clientSecret: xxxxx
      tenantId: xxxxx
    polling:
      initialInterval: 30s
      minInterval: 5s
      maxInterval: 300s
      backoffMultiplier: 2.0
```

## References

1. [Drasi Platform Repository](https://github.com/drasi-project/drasi-platform)
2. [Drasi .NET Source SDK](https://github.com/drasi-project/drasi-platform/tree/main/sources/sdk/dotnet)
3. [Microsoft Dataverse Web API Reference](https://docs.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview)
4. [Azure Managed Identity Documentation](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
5. [Dataverse Change Tracking](https://docs.microsoft.com/en-us/power-apps/developer/data-platform/use-change-tracking-synchronize-data-external-systems)
6. [Issue #338: Update the Dataverse Source](https://github.com/drasi-project/drasi-platform/issues/338)
