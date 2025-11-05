# Kubernetes RBAC Policy Engine with OPA

A Proof-of-Concept Kubernetes RBAC (Role-Based Access Control) and security policy engine built with Open Policy Agent (OPA) using the declarative Rego language.

## Project Overview

This POC demonstrates how to implement a comprehensive Kubernetes policy engine that enforces:

1. **Role-Based Access Control (RBAC)** - Fine-grained user permissions for Kubernetes resources
2. **Security Policies** - Pod security constraints including privileged container restrictions, host access controls, and image policy enforcement
3. **Namespace Isolation** - Users can only access resources within their assigned namespaces
4. **Audit Logging** - Decision logging for compliance and debugging

## Features

### RBAC Features
- **Multiple Roles**: admin, editor, viewer, developer
- **Resource-Specific Permissions**: Different roles have different CRUD permissions on pods, deployments, services, configmaps, and secrets
- **Namespace Scoping**: Non-admin users are restricted to their assigned namespace
- **Permission Aggregation**: Users with multiple roles get combined permissions
- **Audit Trail**: All access decisions are logged with user, resource, and action information

### Security Policy Features
- **Privileged Container Detection**: Prevents containers from running in privileged mode
- **Host Access Restrictions**: Prevents hostNetwork, hostPID, and hostIPC settings
- **Root User Prevention**: Prevents containers from running as UID 0
- **Image Registry Policy**: Only allows images from approved registries (docker.io, gcr.io, quay.io, registry.example.com)
- **Security Report Generation**: Provides detailed violation reports for policy violations

## Project Structure

```
k8s_rbac_opa/
├── policies/
│   ├── utils.rego              # Utility functions for RBAC logic
│   ├── rbac.rego               # Main RBAC policy and decision logic
│   └── security.rego           # Security policy enforcement
├── data/
│   ├── roles.json              # Role definitions with permissions
│   └── users.json              # User assignments and namespace bindings
├── tests/
│   ├── rbac_test.rego          # RBAC policy unit tests
│   └── security_test.rego      # Security policy unit tests
├── examples/
│   ├── pod.json                # Pod examples with expected policy decisions
│   └── deployment.json         # Deployment examples with expected policy decisions
├── claude.md                   # Development best practices
├── project_summary.md          # Project requirements
└── README.md                   # This file
```

## User Roles and Permissions

### Admin Role
- **Access**: All resources across all namespaces
- **Permissions**: Full CRUD access to all resources

### Editor Role
- **Access**: Assigned namespace only
- **Permissions**:
  - Pods: create, read, update, delete
  - Deployments: create, read, update, delete
  - Services: create, read, update, delete
  - ConfigMaps: create, read, update, delete
  - Secrets: read only

### Developer Role
- **Access**: Assigned namespace only
- **Permissions**:
  - Pods: create, read, update, delete
  - Deployments: create, read, update, delete
  - Services: read only
  - ConfigMaps: read, create

### Viewer Role
- **Access**: Assigned namespace only
- **Permissions**:
  - Pods: read only
  - Deployments: read only
  - Services: read only
  - ConfigMaps: read only

## Sample Users

- **alice** - Admin, all namespaces
- **bob** - Editor, production namespace
- **charlie** - Developer, staging namespace
- **diana** - Viewer, production namespace
- **eve** - Developer + Viewer, development namespace
- **frank** - Editor, staging namespace

## RBAC Policy Logic

### Allow Decision
A request is allowed if:
1. The user exists in the user database
2. The user has access to the requested namespace (either admin or assigned namespace)
3. The user's roles contain a permission for the requested resource and action

### Admin Override
Users with the "admin" role bypass all other checks and are allowed any action.

### Deny Reason
When access is denied, a detailed reason is provided explaining why:
- User not found
- No access to namespace
- Missing required permission

### Audit Log
Each request generates an audit log entry including:
- User identity
- Requested resource and action
- Target namespace
- User's roles
- Allow/deny decision

## Security Policy Logic

### Security Check
A pod passes security validation if:
1. ✓ No privileged containers
2. ✓ hostNetwork is false
3. ✓ hostPID is false
4. ✓ hostIPC is false
5. ✓ No containers running as UID 0 (root)
6. ✓ All container images are from approved registries

### Approved Image Registries
- `docker.io/` - Docker Hub
- `gcr.io/` - Google Container Registry
- `quay.io/` - Quay.io
- `registry.example.com/` - Internal registry

### Security Violations
When a pod fails security validation, violations are reported including:
- `privileged_containers_detected`
- `host_access_detected`
- `root_user_detected`
- `unapproved_image_detected`

### Security Report
Security violations generate a detailed report including:
- Pod name and namespace
- Overall security pass/fail status
- List of violations
- Details on specific issues (e.g., which containers are privileged)

## Usage Examples

### Example 1: Admin Creates Deployment
```json
{
  "request": {
    "user": "alice",
    "resource": "deployments",
    "action": "create",
    "namespace": "production"
  }
}
```
**Result**: ✓ Allowed (admin override)

### Example 2: Viewer Cannot Create
```json
{
  "request": {
    "user": "diana",
    "resource": "pods",
    "action": "create",
    "namespace": "production"
  }
}
```
**Result**: ✗ Denied (viewer role has read-only permission)

### Example 3: Namespace Isolation
```json
{
  "request": {
    "user": "bob",
    "resource": "pods",
    "action": "read",
    "namespace": "staging"
  }
}
```
**Result**: ✗ Denied (bob only has access to production namespace)

### Example 4: Security Policy - Privileged Container
```json
{
  "pod": {
    "metadata": {"name": "privileged-pod"},
    "spec": {
      "containers": [
        {
          "securityContext": {
            "privileged": true
          }
        }
      ]
    }
  }
}
```
**Result**: ✗ Security check failed (privileged_containers_detected)

### Example 5: Security Policy - Approved Image
```json
{
  "pod": {
    "spec": {
      "containers": [
        {
          "image": "gcr.io/myproject/app:v1.0"
        }
      ]
    }
  }
}
```
**Result**: ✓ Security check passed (approved registry)

## Testing

The project includes comprehensive test suites for both RBAC and security policies:

### RBAC Tests
Tests cover:
- Admin user access
- Viewer permission restrictions
- Namespace isolation
- Unknown user handling
- Audit log generation

### Security Policy Tests
Tests cover:
- Privileged container detection
- Host access violations (network, PID, IPC)
- Root user detection
- Image policy violations
- InitContainer checks
- Multi-violation scenarios

### Running Tests

To run OPA tests locally, use the OPA CLI:

```bash
# Run all tests
opa test ./tests -v

# Run specific test file
opa test ./tests/rbac_test.rego -v

```

## Policy File Descriptions

### utils.rego
Contains helper functions used by both RBAC and security policies:
- `user_has_role(user, role)` - Check if user has a specific role
- `user_roles(user)` - Get all roles for a user
- `can_access_namespace(user, namespace)` - Check namespace access
- `role_permissions(role)` - Get permissions for a role
- `user_permissions(user)` - Get all permissions for a user
- `user_can_perform_action(...)` - Check if user can perform an action

### rbac.rego
Main RBAC policy file:
- `allow` - Default deny, only allow if permission check passes
- `deny[reason]` - Provide detailed deny reasons
- `user_info` - User context information
- `audit_log` - Audit trail for all requests

### security.rego
Security policy enforcement:
- `security_pass` - Overall security validation result
- `security_violations` - List of detected violations
- `security_report` - Detailed security findings
- Individual check rules for each constraint

## Extending the Policies

### Adding a New Role
1. Add role definition to `data/roles.json`:
```json
{
  "custom_role": {
    "name": "Custom Role",
    "description": "Role description",
    "permissions": [
      {"resource": "pods", "action": "read"}
    ]
  }
}
```

2. Assign role to users in `data/users.json`:
```json
{
  "user_name": {
    "roles": ["custom_role"]
  }
}
```

### Adding a New Security Policy
1. Add policy rule to `security.rego`:
```rego
new_policy_violation {
  # Check condition
  condition_check
}

# Add to security_pass rule
security_pass {
  # ... existing checks
  not new_policy_violation
}
```

2. Add corresponding test to `tests/security_test.rego`

### Adding New Resources
1. Update role permissions in `data/roles.json`
2. Update RBAC logic in `policies/rbac.rego` if needed
3. Add corresponding tests

## Best Practices

- **Least Privilege**: Assign users the minimum required roles
- **Regular Audits**: Review audit logs for suspicious activity
- **Policy Version Control**: Keep policies in version control
- **Namespace Segmentation**: Use namespace boundaries for tenancy
- **Image Registry Control**: Maintain tight control over approved registries
- **Testing**: Always test policy changes with comprehensive test cases


## Future Enhancements

This POC can be extended with:
- Resource quota policies (`quotas.rego`)
- Network policy enforcement
- RBAC audit log aggregation
- Webhook integration with Kubernetes API server
- Custom policy authoring UI
- Performance monitoring
- Advanced ABAC patterns

## References

- [Open Policy Agent Documentation](https://www.openpolicyagent.org/)
- [OPA/Rego Language Guide](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

## License
