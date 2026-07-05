# <Module Name> RD

## 1. Purpose

Describe what this module does and why it exists.

## 2. Scope

### In Scope

- List included features.

### Out of Scope

- List items intentionally not included.

## 3. Actors and Roles

| Role | Access / Responsibility |
| --- | --- |
| ADMIN | TBD |

## 4. Functional Requirements

| ID | Requirement | Priority |
| --- | --- | --- |
| FR-001 | TBD | Must |

## 5. API Requirements

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/api/...` | TBD | TBD |

### Request Body

```json
{
  "field": "value"
}
```

### Success Response

```json
{
  "message": "Success"
}
```

### Error Responses

| Status | Scenario | Message / Rule |
| --- | --- | --- |
| 400 | Validation failure | TBD |
| 401 | Invalid authentication | TBD |
| 403 | Forbidden role | TBD |

## 6. Validation Rules

- TBD

## 7. Database Requirements

| Table | Operation | Notes |
| --- | --- | --- |
| TBD | Insert / Update / Read | TBD |

## 8. Security Rules

- Authentication requirements.
- Authorization requirements.
- Token/session rules, if any.
- Sensitive fields that must not be logged or returned.

## 9. Business Rules

- TBD

## 10. Development Notes

- Service/class names.
- Existing utilities to reuse.
- Integration points with other modules.

## 11. Test Checklist

### Unit Tests

- TBD

### Integration / API Tests

- TBD

### Negative Tests

- TBD

## 12. Open Questions

- TBD
