# LedgerApp — Complete Engineering & Design Standards

**Version 2.0 · Accounting Software for Indian Businesses**  
`ASP.NET Core 8 · C# 12 · Clean Architecture · CQRS · DDD · Multi-Company · Keyboard-First`

> **How to use this document:**  
> Read it once fully when you join the project. Then paste **Section 12 (AI Context Block)** at the top of every AI coding session. Update this document in the same PR as any convention change — never separately.

---

## Table of Contents

1. [Solution Architecture](#1-solution-architecture)
2. [Keyboard-First Navigation](#2-keyboard-first-navigation)
3. [Reactive State — Single Source of Truth](#3-reactive-state--single-source-of-truth)
4. [Multi-Company & Data Portability](#4-multi-company--data-portability)
5. [Decimal & Money Handling](#5-decimal--money-handling)
6. [Theme System — Light & Dark](#6-theme-system--light--dark)
7. [Typography & Accessibility](#7-typography--accessibility)
8. [Localization — English, Hindi, Punjabi](#8-localization--english-hindi-punjabi)
9. [Coding Standards](#9-coding-standards)
10. [Testing Protocols](#10-testing-protocols)
11. [Auto-Update System](#11-auto-update-system)
12. [Security, Auth & Error Handling](#12-security-auth--error-handling)
13. [AI Prompt Context Block ← paste this every session](#13-ai-prompt-context-block)
14. [Master Scaffold Prompt ← start project here](#14-master-scaffold-prompt)
15. [Quick Reference Card](#15-quick-reference-card)

---

## 1. Solution Architecture

### 1.1 Four-Layer Rule

Outer layers depend on inner layers. **Never the reverse.** The Domain layer has zero infrastructure dependencies — it is pure C# business logic testable with no database, no HTTP, no EF Core.

```
┌─────────────────────────────────────────────────────────────────┐
│  API / UI Layer        Depends on Application only              │
│  Minimal API endpoints, Blazor/React components, Middleware      │
├─────────────────────────────────────────────────────────────────┤
│  Infrastructure Layer  Depends on Application only              │
│  EF Core, Repositories, JWT, Serilog, Email, SignalR, Storage   │
├─────────────────────────────────────────────────────────────────┤
│  Application Layer     Depends on Domain only                   │
│  CQRS Commands/Queries, MediatR Handlers, DTOs, Validators      │
├─────────────────────────────────────────────────────────────────┤
│  Domain Layer          ZERO external dependencies               │
│  Entities, Value Objects, Domain Events, Interfaces, Exceptions │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Project Folder Structure

```
LedgerApp.sln
├── src/
│   ├── LedgerApp.Domain/
│   │   ├── Entities/              # Invoice, LedgerEntry, Account, Payment, Company
│   │   ├── ValueObjects/          # Money, AccountCode, TaxRate, FiscalYear
│   │   ├── Events/                # Domain events as C# records
│   │   ├── Interfaces/            # IRepository<T>, IUnitOfWork, ICompanyContext
│   │   ├── Exceptions/            # DomainException, NotFoundException, etc.
│   │   └── Constants/             # GST_RATES, MAX_LINE_ITEMS, SUPPORTED_LOCALES
│   │
│   ├── LedgerApp.Application/
│   │   ├── Features/
│   │   │   ├── Invoices/          # Commands/, Queries/, DTOs/
│   │   │   ├── LedgerEntries/
│   │   │   ├── Payments/
│   │   │   ├── Companies/         # Multi-company management
│   │   │   ├── DataPortability/   # Export, Import commands
│   │   │   └── Reports/
│   │   └── Common/
│   │       ├── Behaviors/         # LoggingBehavior, ValidationBehavior, TransactionBehavior
│   │       └── Interfaces/        # ICurrentUserService, ICompanyContext
│   │
│   ├── LedgerApp.Infrastructure/
│   │   ├── Persistence/           # AppDbContext, Configurations/, Migrations/
│   │   ├── Repositories/
│   │   ├── Services/              # JWT, Email, PDF, DataExport, AutoUpdate
│   │   └── SignalR/               # AccountingHub
│   │
│   └── LedgerApp.API/
│       ├── Endpoints/             # One file per feature group
│       ├── Middleware/            # ExceptionHandler, CorrelationId, RateLimiting
│       └── Hubs/                  # SignalR hubs
│
└── tests/
    ├── LedgerApp.Tests.Unit/       # xUnit + NSubstitute + FluentAssertions
    ├── LedgerApp.Tests.Integration/ # TestContainers + real SQL Server
    └── LedgerApp.Tests.E2E/        # Playwright — keyboard-only tests
```

### 1.3 CQRS Pattern

Every operation is a **Command** (changes state) or a **Query** (reads state). They never mix.

```csharp
// COMMAND — changes data, one file per use case
public record CreateInvoiceCommand(
    Guid CustomerId,
    List<LineItemDto> Items,
    DateOnly DueDate,
    Guid TaxRateId
) : IRequest<InvoiceResponse>;

// QUERY — reads data, never changes anything
public record GetInvoiceByIdQuery(Guid Id) : IRequest<InvoiceDetailDto>;

public record GetInvoicesPagedQuery(
    InvoiceFilter Filter,
    int Page,
    int PageSize
) : IRequest<PagedResult<InvoiceSummaryDto>>;

// HANDLER — one handler per command/query
// Inject IUnitOfWork — NEVER AppDbContext directly
// Delegate business rules to domain entities — NEVER put logic here
public class CreateInvoiceCommandHandler
    : IRequestHandler<CreateInvoiceCommand, InvoiceResponse>
{
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICompanyContext _companyContext;

    public async Task<InvoiceResponse> Handle(
        CreateInvoiceCommand request,
        CancellationToken ct)
    {
        // Orchestrate — don't calculate
        var invoice = Invoice.Create(
            request.CustomerId,
            _companyContext.CompanyId,
            request.Items.Select(LineItem.From).ToList()
        );
        await _unitOfWork.Invoices.AddAsync(invoice);
        await _unitOfWork.SaveChangesAsync(ct);
        return InvoiceResponse.From(invoice);
    }
}
```

### 1.4 MediatR Pipeline Behaviors (applied in this order)

```csharp
// 1. LoggingBehavior    — request name, user ID, elapsed ms, company ID
// 2. ValidationBehavior — runs FluentValidation, aggregates ALL field errors
// 3. TransactionBehavior — wraps commands in DB transaction,
//                          dispatches domain events AFTER commit
```

---

## 2. Keyboard-First Navigation

> **Context:** Our primary users are accountants trained on DOS-based software (Tally, etc.). They never use a mouse. Every single action must be reachable by keyboard alone.

### 2.1 The Focus Engine — Apple TV Spatial Model

The UI is divided into named **Focus Zones**. Focus moves in four spatial directions: Up, Down, Left, Right. Enter activates. Escape goes back one level (pops the focus history stack). There is no need to Tab through unrelated elements to reach something nearby.

```
┌──────────────┐   ←→   ┌────────────────────────────────┐
│  Sidebar Nav │         │  Toolbar / Action Bar          │
│  (FocusZone) │         ├────────────────────────────────┤
│              │         │  Data Grid / Form Fields       │
│              │         │  (FocusZone — bidirectional)   │
│              │         ├────────────────────────────────┤
│              │         │  Footer / Status Bar           │
└──────────────┘         └────────────────────────────────┘
```

```tsx
// FocusZone component — wrap every navigable region
<FocusZone
  id="invoice-grid"
  direction="bidirectional"       // Up/Down/Left/Right
  onKeyDown={handleGridNavigation}
  defaultActiveElement="#first-row"
>
  {rows.map(row => <GridRow focusable key={row.id} />)}
</FocusZone>
```

### 2.2 Global Keyboard Shortcuts

| Key | Action | Available From |
|-----|--------|----------------|
| `F1` | Context-sensitive help | Anywhere |
| `F2` | Edit selected item | Any grid/list |
| `F3` / `Ctrl+F` | Open search/filter | Any screen |
| `F4` | Open lookup / dropdown | Lookup fields |
| `F5` | Refresh / reload data | Any screen |
| `F8` | Mark voucher as approved | Voucher screens |
| `F9` | Post / Save current entry | All forms |
| `F10` | Open main menu | Anywhere |
| `F12` | Print / Export current view | Any screen |
| `Ctrl+N` | New record | Any list screen |
| `Ctrl+S` | Save current form | Any form |
| `Ctrl+Z` | Undo last action | Forms |
| `Ctrl+D` | Duplicate selected record | Any list |
| `Ctrl+P` | Print current record | Detail screens |
| `Ctrl+E` | Export current view | Any list/report |
| `Ctrl+I` | Import data file | List screens |
| `Ctrl+W` | Switch company | Anywhere |
| `Alt+F4` | Close current panel/modal | Modals, panels |
| `Escape` | Cancel / go back one level | Anywhere |
| `Enter` | Confirm / activate focused item | Anywhere |
| `↑ ↓ ← →` | Navigate grid rows and columns | All grids |
| `Tab` / `Shift+Tab` | Move between form fields | All forms |
| `Page Up/Down` | Scroll grid by page | All grids |
| `Home` / `End` | First / last item in grid | All grids |
| `Ctrl+Home` | First page of results | Paginated grids |
| `Ctrl+End` | Last page of results | Paginated grids |

> **Rule:** Always include the keyboard shortcut in button labels.  
> Example: `Save (F9)`, `New Invoice (Ctrl+N)`, `Post Entry (F9)`

### 2.3 Focus Visual Design — Dashed Border

```css
/* ── Global focus styles ──────────────────────────────────── */

/* Remove browser default — we control it entirely */
*:focus { outline: none; }

/* The canonical focus indicator — dashed, high contrast */
.focusable:focus-visible,
[data-focus-zone] *:focus-visible {
  outline: 2px dashed var(--focus-ring-color);
  outline-offset: 3px;
  border-radius: 2px;
}

/* Light theme */
[data-theme="light"] {
  --focus-ring-color: #1B3A5C;   /* deep navy — high contrast on white */
}

/* Dark theme */
[data-theme="dark"] {
  --focus-ring-color: #60A5FA;   /* bright blue — visible on dark background */
}

/* Grid row — highlight full row when it contains focus */
.grid-row:focus-within {
  background: var(--bg-row-focused);
  outline: 2px dashed var(--focus-ring-color);
  outline-offset: -2px;
}

/* NEVER use outline: none without providing a replacement */
```

> **Rule:** Every focusable element must have a visible focus indicator. Run `axe-core` on every PR. Zero focus-related violations allowed to merge.

### 2.4 Grid Keyboard Handler

```typescript
// Attach to every data grid component
function handleGridKeyDown(
  e: KeyboardEvent,
  currentRow: number,
  currentCol: number,
  rowCount: number,
  colCount: number
) {
  const PAGE_SIZE = 20;

  switch (e.key) {
    case 'ArrowUp':    focusCell(currentRow - 1, currentCol);                    break;
    case 'ArrowDown':  focusCell(currentRow + 1, currentCol);                    break;
    case 'ArrowLeft':  focusCell(currentRow, currentCol - 1);                    break;
    case 'ArrowRight': focusCell(currentRow, currentCol + 1);                    break;
    case 'Enter':      openCellEditor(currentRow, currentCol);                   break;
    case 'Escape':     cancelEdit(); restorePreviousFocus();                     break;
    case 'Tab':        focusCell(currentRow, currentCol + 1);                    break;
    case 'PageUp':     focusRow(Math.max(0, currentRow - PAGE_SIZE));            break;
    case 'PageDown':   focusRow(Math.min(rowCount - 1, currentRow + PAGE_SIZE)); break;
    case 'Home':       focusRow(0);                                              break;
    case 'End':        focusRow(rowCount - 1);                                   break;
    case 'Delete':     clearCellValue(currentRow, currentCol);                   break;
    case 'F2':         openCellEditor(currentRow, currentCol);                   break;
    case 'F9':         saveAndMoveToNextRow(currentRow);                         break;
  }
}
```

### 2.5 KeyboardNavigationService

```csharp
// C# / Blazor server-side service
public interface IKeyboardNavigationService
{
    /// <summary>
    /// Registers a focusable zone. Zones know their spatial neighbors.
    /// </summary>
    void RegisterZone(FocusZone zone);

    /// <summary>
    /// Routes focus in a spatial direction from the currently active zone.
    /// </summary>
    Task MoveFocusAsync(FocusDirection direction);

    /// <summary>
    /// Pushes current focus to history stack and moves to new target.
    /// Escape key pops the stack and restores.
    /// </summary>
    Task PushFocusAsync(string zoneId);

    /// <summary>
    /// Pops focus history and restores previous location.
    /// Called on every Escape key press.
    /// </summary>
    Task PopFocusAsync();
}

public enum FocusDirection { Up, Down, Left, Right }
```

---

## 3. Reactive State — Single Source of Truth

> **Rule:** No component holds its own copy of server data. All data comes from the central store. Components are projections of state — not owners of state.

### 3.1 State Architecture (Blazor — Fluxor)

```csharp
// 1. STATE RECORD — immutable, one per domain feature
public record InvoiceState
{
    public IReadOnlyList<InvoiceSummaryDto> Invoices { get; init; } = [];
    public InvoiceDetailDto?               SelectedInvoice { get; init; }
    public bool                            IsLoading { get; init; }
    public string?                         ErrorMessage { get; init; }
    public InvoiceFilter                   ActiveFilter { get; init; } = new();
}

// 2. ACTIONS — plain records describing what happened
public record LoadInvoicesAction(InvoiceFilter Filter);
public record InvoicesLoadedAction(IReadOnlyList<InvoiceSummaryDto> Invoices);
public record InvoiceCreatedAction(InvoiceSummaryDto Invoice);
public record RemoteUpdateReceivedAction(string EntityType, JsonElement Payload);

// 3. REDUCER — pure function, no side effects
public static InvoiceState Reduce(InvoiceState state, object action) => action switch
{
    LoadInvoicesAction    _  => state with { IsLoading = true, ErrorMessage = null },
    InvoicesLoadedAction  a  => state with { Invoices = a.Invoices, IsLoading = false },
    InvoiceCreatedAction  a  => state with { Invoices = [a.Invoice, ..state.Invoices] },
    _                        => state
};

// 4. EFFECTS — side effects only, dispatch actions on result
[EffectMethod]
public async Task HandleLoad(LoadInvoicesAction action, IDispatcher dispatcher)
{
    var result = await _mediator.Send(new GetInvoicesPagedQuery(action.Filter));
    dispatcher.Dispatch(new InvoicesLoadedAction(result.Items));
}
```

### 3.2 State Architecture (React — Zustand)

```typescript
// One store per domain — verb-noun actions, no setX generics
interface InvoiceStore {
  // State
  invoices: InvoiceSummary[];
  selectedInvoice: InvoiceDetail | null;
  isLoading: boolean;
  error: string | null;
  filter: InvoiceFilter;

  // Actions — verb-noun naming
  loadInvoices: (filter: InvoiceFilter) => Promise<void>;
  selectInvoice: (id: string) => void;
  addInvoice: (invoice: InvoiceSummary) => void;
  clearError: () => void;
}

export const useInvoiceStore = create<InvoiceStore>()(
  devtools(
    (set) => ({
      invoices: [],
      selectedInvoice: null,
      isLoading: false,
      error: null,
      filter: defaultFilter,

      loadInvoices: async (filter) => {
        set({ isLoading: true, error: null });
        try {
          const data = await invoiceService.getAll(filter);
          set({ invoices: data.items, isLoading: false });
        } catch (e) {
          set({ error: getErrorMessage(e), isLoading: false });
        }
      },
      addInvoice: (invoice) =>
        set((s) => ({ invoices: [invoice, ...s.invoices] })),
    }),
    { name: 'InvoiceStore' }  // devtools name
  )
);

// Selector hooks — shallow equality to prevent unnecessary re-renders
export const usePendingInvoices = () =>
  useInvoiceStore(
    (s) => s.invoices.filter(i => i.status === 'Pending'),
    shallow
  );
```

### 3.3 Real-Time Updates via SignalR

```csharp
// Server — broadcast after any successful command
public class AccountingHub : Hub
{
    /// <summary>
    /// Broadcasts an entity change to all members of the same company.
    /// Called from TransactionBehavior after commit.
    /// </summary>
    public async Task BroadcastUpdate(
        string companyId,
        string entityType,
        object payload)
        => await Clients.Group(companyId)
                        .SendAsync("EntityUpdated", entityType, payload);
}
```

```typescript
// Client — subscribe once, update store everywhere
hubConnection.on('EntityUpdated', (entityType: string, payload: unknown) => {
  // Dispatch to store — all subscribed views re-render automatically
  switch (entityType) {
    case 'Invoice':  useInvoiceStore.getState().addInvoice(payload as InvoiceSummary); break;
    case 'Payment':  usePaymentStore.getState().addPayment(payload as PaymentSummary); break;
    case 'Ledger':   useLedgerStore.getState().addEntry(payload as LedgerEntry);       break;
  }
  // Quiet notification: "Updated by Ramesh (3:42 PM)" — bottom status bar, not a popup
  showStatusNotification(`Data updated by ${payload.updatedBy}`);
});
```

---

## 4. Multi-Company & Data Portability

### 4.1 Company Isolation

Every piece of data belongs to a company. Isolation is enforced at every layer.

```csharp
// ICompanyContext — injected into ALL handlers and repositories
public interface ICompanyContext
{
    Guid   CompanyId { get; }
    string CompanyName { get; }
    string FiscalYearStart { get; }  // "04-01" for April 1st (Indian FY)
    string DefaultCurrency { get; }  // "INR"
    string DefaultLocale { get; }    // "en-IN" | "hi-IN" | "pa-IN"
}

// Repository base — ALL queries automatically scoped to current company
public abstract class CompanyScopedRepository<T> where T : CompanyEntity
{
    protected IQueryable<T> BaseQuery =>
        _context.Set<T>()
                .Where(e => e.CompanyId == _companyContext.CompanyId);
    // Never use _context.Set<T>() directly — always BaseQuery
}

// EF Core global filter — safety net if a developer bypasses BaseQuery
modelBuilder.Entity<Invoice>()
    .HasQueryFilter(i => i.CompanyId == _companyContext.CompanyId);
```

### 4.2 Export / Import Format

```jsonc
// File: CompanyName_YYYYMMDD.ledger  (JSON, gzip compressed)
{
  "schemaVersion": "2.0",
  "exportedAt": "2025-04-01T10:30:00Z",
  "exportedBy": "admin@abctraders.com",
  "company": {
    "id": "...",
    "name": "ABC Traders Pvt. Ltd.",
    "gstNumber": "07AABCU9603R1ZX",
    "pan": "AABCU9603R"
  },
  "chartOfAccounts": [ /* ... */ ],
  "fiscalYears": [
    {
      "year": "2024-25",
      "openingBalances": [ /* ... */ ],
      "ledgerEntries": [ /* ... */ ],
      "invoices": [ /* ... */ ],
      "payments": [ /* ... */ ]
    }
  ],
  "checksum": "sha256:abc123..."  // Verified on import to detect corruption
}
```

```csharp
// Export command — triggered by Ctrl+E
public record ExportCompanyDataCommand(
    Guid      CompanyId,
    DateOnly  From,
    DateOnly  To,
    string[]  Formats   // "ledger", "csv", "xlsx"
) : IRequest<ExportResult>;

// Import command — validates, shows dry-run preview, then commits
public record ImportCompanyDataCommand(
    Stream     FileStream,
    ImportMode Mode       // CreateNew | MergeExisting | ReplaceExisting
) : IRequest<ImportResult>;
```

| Concern | Rule |
|---------|------|
| Export formats | `.ledger` (native full fidelity), `.csv` (per table, Tally compatible), `.xlsx` |
| Import validation | Schema version check → checksum → duplicate detection → dry-run preview → commit |
| Keyboard shortcuts | `Ctrl+E` export, `Ctrl+I` import, `Ctrl+W` switch company |
| Audit | Every export/import logged to `AuditLog` with user, timestamp, record counts |
| Company switch | `Ctrl+W` opens switcher. No logout. State fully isolated per company. |

---

## 5. Decimal & Money Handling

> **One paisa matters.** In Indian accounting, 1 paisa discrepancy in a ledger is unacceptable.

### 5.1 The Money Value Object — always use it

```csharp
/// <summary>
/// Represents a monetary amount with currency.
/// Always use this — never raw decimal or double for any financial value.
/// </summary>
public sealed record Money
{
    public decimal Amount   { get; }
    public string  Currency { get; }  // ISO 4217: "INR"

    private Money(decimal amount, string currency)
    {
        // Round BEFORE validation — prevents rounding edge cases
        var rounded = Math.Round(amount, 2, MidpointRounding.AwayFromZero);

        if (rounded < 0)
            throw new DomainException("Money amount cannot be negative. " +
                "Use debit/credit direction instead of negative sign.");

        Amount   = rounded;
        Currency = currency.ToUpperInvariant();
    }

    public static Money InrOf(decimal amount) => new(amount, "INR");
    public static Money Zero                  => new(0m,     "INR");

    public static Money operator +(Money a, Money b)
    {
        EnsureSameCurrency(a, b);
        return new(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator -(Money a, Money b)
    {
        EnsureSameCurrency(a, b);
        return new(a.Amount - b.Amount, a.Currency);
    }

    /// <summary>
    /// Display in Indian number format: ₹1,23,456.78
    /// </summary>
    public string Display(string locale = "en-IN")
        => Amount.ToString("C2", new CultureInfo(locale));

    private static void EnsureSameCurrency(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new DomainException(
                $"Currency mismatch: cannot operate on {a.Currency} and {b.Currency}.");
    }
}
```

### 5.2 Decimal Rules

| Rule | Detail |
|------|--------|
| **Rounding method** | `MidpointRounding.AwayFromZero` always. Never `ToEven` (banker's rounding causes issues). |
| **Decimal places** | 2 decimal places for INR (paise). Never truncate — always round. |
| **DB column type** | `decimal(18, 2)` in SQL Server. Never `float`, `real`, or `money` type. |
| **EF Core mapping** | `.HasColumnType("decimal(18,2)")` in Fluent API on every Money column. |
| **Display format** | `CultureInfo("en-IN")` → Indian system: `₹1,23,456.78` |
| **Zero check** | `money.Amount == 0m` — always use decimal literal `m` suffix. |
| **GST calculation** | Calculate on taxable amount BEFORE discounts. Use `TaxRate.Calculate(Money)` — never inline `0.18m`. |
| **float / double ban** | These types are **banned** for any financial field. Add a Roslyn analyzer rule. |

```csharp
// ✅ CORRECT
var taxableAmount = Money.InrOf(10_000m);
var gst           = TaxRate.Gst18.Calculate(taxableAmount);  // ₹1,800.00
var total         = taxableAmount + gst;                      // ₹11,800.00
var display       = total.Display("en-IN");                   // "₹11,800.00"

// ✅ CORRECT rounding
decimal tax = Math.Round(amount * 0.18m, 2, MidpointRounding.AwayFromZero);

// ❌ WRONG — never do this
double tax  = (double)amount * 0.18;   // float precision errors accumulate
decimal tax2 = amount * 0.18m;          // missing rounding — paisa discrepancy
string bad   = amount.ToString("C");    // Western format: ₹1,000.00 not ₹1,000.00
```

### 5.3 Indian GST Slabs

```csharp
public static class GstSlabs
{
    public static readonly TaxRate Exempt = TaxRate.Of(0m);    // 0%  — basic food, etc.
    public static readonly TaxRate Slab5  = TaxRate.Of(5m);    // 5%  — essential goods
    public static readonly TaxRate Slab12 = TaxRate.Of(12m);   // 12% — processed food, etc.
    public static readonly TaxRate Slab18 = TaxRate.Of(18m);   // 18% — most services/goods
    public static readonly TaxRate Slab28 = TaxRate.Of(28m);   // 28% — luxury goods

    // For intra-state: split as CGST (half) + SGST (half)
    // For inter-state: apply full rate as IGST
}
```

---

## 6. Theme System — Light & Dark

> **Color carries meaning.** Teal = settled/paid. Amber = pending. Red = overdue/error. Green = income/credit. These meanings **never change** — so support calls over phone are easy: "press the amber button."

### 6.1 Light Theme — Pastel Cool

```css
[data-theme="light"] {
  /* Backgrounds */
  --bg-base:          #F8FAFC;  /* page bg — cool off-white, never pure white */
  --bg-surface:       #FFFFFF;  /* cards, panels, modals */
  --bg-sidebar:       #EFF6FF;  /* navigation sidebar — very light blue */
  --bg-row-focused:   #DBEAFE;  /* focused grid row */
  --bg-row-hover:     #F1F5F9;  /* hovered grid row */

  /* Text */
  --text-primary:     #1E293B;  /* headings, labels — deep slate */
  --text-secondary:   #475569;  /* descriptions, helper text */
  --text-muted:       #94A3B8;  /* placeholders, disabled */

  /* Semantic accents */
  --accent-blue:      #3B82F6;  /* primary actions, links */
  --accent-teal:      #0D9488;  /* posted, approved, paid, credit */
  --accent-amber:     #F59E0B;  /* pending, draft, unreconciled */
  --accent-red:       #EF4444;  /* overdue, error, debit */
  --accent-green:     #22C55E;  /* income, credit, positive balance */
  --accent-purple:    #7C3AED;  /* admin-only features */

  /* Money */
  --money-debit:      #DC2626;  /* all debit / expense amounts */
  --money-credit:     #16A34A;  /* all credit / income amounts */

  /* Borders */
  --border-default:   #E2E8F0;
  --border-focus:     #1B3A5C;
  --focus-ring-color: #1B3A5C;  /* dashed focus border */
}
```

### 6.2 Dark Theme — Pastel Midnight

```css
[data-theme="dark"] {
  --bg-base:          #0F172A;  /* deep navy — not pure black */
  --bg-surface:       #1E293B;  /* cards, panels */
  --bg-sidebar:       #162032;
  --bg-row-focused:   #1E3A5F;
  --bg-row-hover:     #263348;

  --text-primary:     #F1F5F9;  /* soft white — not pure #FFF */
  --text-secondary:   #94A3B8;
  --text-muted:       #475569;

  --accent-blue:      #60A5FA;  /* lighter for dark background */
  --accent-teal:      #2DD4BF;
  --accent-amber:     #FCD34D;
  --accent-red:       #FCA5A5;  /* pastel red — not harsh */
  --accent-green:     #86EFAC;  /* pastel green */
  --accent-purple:    #A78BFA;

  --money-debit:      #FCA5A5;  /* soft red on dark */
  --money-credit:     #86EFAC;  /* soft green on dark */

  --border-default:   #334155;
  --border-focus:     #60A5FA;
  --focus-ring-color: #60A5FA;
}
```

> **Rule:** `NEVER` hardcode a color hex value in any component. Always use CSS variables. This is what makes theme switching work in one line.

---

## 7. Typography & Accessibility

### 7.1 Font Stack

| Context | Font | Min Size | Why |
|---------|------|----------|-----|
| Financial amounts, account codes | **JetBrains Mono** | 16px | Digits never confused: `0` vs `O`, `1` vs `l`, columns align |
| Voucher numbers, reference codes | JetBrains Mono | 15px | Auditor-friendly precision reading |
| Navigation, headings, labels | **Inter** or **Nunito** | 16px | Warm, friendly, highly legible |
| Body text, descriptions | Inter or Nunito | 15px | Comfortable for long sessions |
| Money input fields | JetBrains Mono | 16px | User can verify each digit |
| Error messages, warnings | Inter Bold | 14px minimum | Must be clearly distinct |
| Print / PDF output | JetBrains Mono (figures) + Calibri (text) | 11pt | Legible on A4 paper |

> **Rule:** Minimum font size is **14px** anywhere in the application. The audience includes senior accountants and business owners. When in doubt, go larger.

### 7.2 Accessibility Rules

| Rule | Standard |
|------|----------|
| Contrast ratio | WCAG AA: 4.5:1 body text, 3:1 large text (18px+) |
| Focus visibility | Dashed 2px outline, 3px offset — see Section 2.3 |
| Screen reader | `aria-label="12,500 Indian Rupees"` on amounts. `aria-label="Status: Paid"` on badges. |
| Error association | `aria-describedby` links error to its field. Never color-only errors. |
| Motion | Respect `prefers-reduced-motion` — disable transitions when set. |
| Zoom | Fully functional at 150% and 200% browser zoom. Test on every release. |

---

## 8. Localization — English, Hindi, Punjabi

### 8.1 Rules

| Rule | Detail |
|------|--------|
| Zero hardcoded strings | Every user-visible string uses a resource key. Zero exceptions. |
| Key format | `Feature.Section.Key` — e.g. `Invoice.Status.Paid`, `Button.Save`, `Error.Required` |
| Default locale | `en-IN` — English with Indian number/date format |
| Date format | `dd/MM/yyyy` across all locales — never `MM/dd/yyyy` |
| Number format | Indian system: `1,00,000.00` — not Western `100,000.00` |
| Locale switching | Per-session without logout. Only labels change — state preserved. |
| Missing keys | Fall back to `en-IN`. Log missing keys as `Warning`. Never show a raw key. |

### 8.2 Resource File Examples

```
Resources/
├── en-IN.resx   # English (default)
├── hi-IN.resx   # Hindi (Devanagari)
└── pa-IN.resx   # Punjabi (Gurmukhi)
```

| Key | English | Hindi | Punjabi |
|-----|---------|-------|---------|
| `Invoice.Total` | Total Amount | कुल राशि | ਕੁੱਲ ਰਕਮ |
| `Invoice.Status.Paid` | Paid | भुगतान | ਭੁਗਤਾਨ |
| `Invoice.Status.Pending` | Pending | लंबित | ਬਕਾਇਆ |
| `Invoice.Status.Overdue` | Overdue | अतिदेय | ਬਕਾਇਆ ਤਾਰੀਖ਼ |
| `Button.Save` | Save (F9) | सहेजें (F9) | ਸੇਵ ਕਰੋ (F9) |
| `Button.New` | New (Ctrl+N) | नया (Ctrl+N) | ਨਵਾਂ (Ctrl+N) |
| `LedgerEntry.Debit` | Dr | डेबिट | ਡੈਬਿਟ |
| `LedgerEntry.Credit` | Cr | क्रेडिट | ਕ੍ਰੈਡਿਟ |

```csharp
// ✅ CORRECT — always use localizer
var message = _localizer["Invoice.Status.Paid"];

// ❌ WRONG — never hardcode
var message = "Invoice has been paid";
```

---

## 9. Coding Standards

### 9.1 Naming Conventions

| What | Convention | Example |
|------|-----------|---------|
| Classes | PascalCase | `InvoiceService`, `TaxCalculator` |
| Interfaces | `I` + PascalCase | `IInvoiceRepository`, `IUnitOfWork` |
| Methods | PascalCase | `CalculateTax()`, `GetInvoiceByIdAsync()` |
| Variables | camelCase | `invoiceTotal`, `taxRate` |
| Constants | `UPPER_SNAKE_CASE` | `GST_RATE_18`, `MAX_LINE_ITEMS` |
| Private fields | `_camelCase` | `_unitOfWork`, `_logger` |
| Booleans | `is`/`has`/`can` prefix | `isPosted`, `hasDiscount`, `canEdit` |
| Async methods | `Async` suffix | `GetInvoicesAsync()`, `SaveAsync()` |
| Commands | Verb + Noun + Command | `CreateInvoiceCommand` |
| Queries | `Get` + Noun + `Query` | `GetInvoiceByIdQuery` |
| Tests | `MethodName_Scenario_ExpectedResult` | `Handle_ValidCommand_CreatesInvoice` |

### 9.2 Comment Standards

```csharp
/// <summary>
/// Calculates GST at the applicable rate for the given taxable amount.
///
/// Per GST Act 2017, Section 15:
/// - Intra-state supply: CGST (half rate) + SGST (half rate)
/// - Inter-state supply: IGST (full rate)
///
/// Always use this method — never hardcode 0.18m or any tax rate inline.
/// </summary>
/// <param name="amount">Pre-tax taxable amount as a Money value object.</param>
/// <param name="isInterState">
///   True for inter-state supply (IGST applies).
///   False for intra-state (CGST + SGST applies).
/// </param>
/// <returns>
///   Tax amount as Money. Always non-negative.
///   Rounded to 2 decimal places using MidpointRounding.AwayFromZero.
/// </returns>
public Money CalculateGst(Money amount, bool isInterState)
{
    // TaxRate value object handles rounding — never apply rate directly to decimal
    var rate = isInterState ? TaxRate.Igst18 : TaxRate.Cgst9;
    return rate.Calculate(amount);
}
```

> **Rule:** Comments explain **WHY**, not WHAT. The code says what. The comment explains the accounting rule, the legal reference, or the non-obvious decision.

### 9.3 File and Method Limits

| Limit | Value | What to do if exceeded |
|-------|-------|------------------------|
| File length | 200 lines | Split into smaller files, add `index.cs` barrel exports |
| Method length | 20 lines | Extract private methods with descriptive names |
| Parameters | 3 max | Use a typed request record/object beyond 3 |
| Nesting depth | 3 levels | Use early returns (guard clauses) |
| Classes per file | 1 | Always — file name must match class name |

### 9.4 Required Documentation Files

| File | Contents |
|------|----------|
| `/ README.md` | Project overview, how to run locally, environment variables, tech stack |
| `/docs/ARCHITECTURE.md` | Architecture decisions, why Clean Architecture, why CQRS |
| `/docs/DOMAIN.md` | Accounting domain glossary in plain developer language |
| `/docs/API.md` | All endpoints, request/response formats, error codes |
| `/docs/CHANGELOG.md` | All changes by version: Added / Changed / Fixed / Deprecated |
| `Features/{Name}/README.md` | What the feature does, what to edit, what NOT to touch |

---

## 10. Testing Protocols

### 10.1 Testing Pyramid

| Layer | Coverage | Tools | Scope |
|-------|----------|-------|-------|
| Unit (70%) | Domain + Application handlers | xUnit, NSubstitute, FluentAssertions | No DB, no HTTP, pure logic |
| Integration (20%) | Repository + DB interactions | TestContainers, real SQL Server | Verify EF mappings, queries |
| E2E (10%) | Critical user journeys | Playwright, keyboard-only | No mouse events allowed |

### 10.2 Unit Test Standards

```csharp
// Test class naming: {ClassUnderTest}Tests
// Test method naming: MethodName_Scenario_ExpectedResult
public class CreateInvoiceCommandHandlerTests
{
    [Fact]
    public async Task Handle_ValidCommand_CreatesInvoiceAndRaisesEvent()
    {
        // ARRANGE — use builder pattern for readable setup
        var command = new InvoiceCommandBuilder()
            .WithCustomer(Guid.NewGuid())
            .WithLineItem("Consulting", Money.InrOf(10_000m), qty: 1)
            .WithTaxRate(TaxRate.Gst18)
            .Build();

        var unitOfWork = Substitute.For<IUnitOfWork>();
        var handler    = new CreateInvoiceCommandHandler(
            unitOfWork,
            Substitute.For<ICompanyContext>()
        );

        // ACT
        var result = await handler.Handle(command, CancellationToken.None);

        // ASSERT — FluentAssertions for readable failure messages
        result.Should().NotBeNull();
        result.Total.Amount.Should().Be(11_800.00m);  // ₹10,000 + 18% GST
        await unitOfWork.Received(1).SaveChangesAsync();
    }

    // ── MANDATORY: Decimal / Paisa edge cases ─────────────────────────────
    [Theory]
    [InlineData(100.005, 100.01)]   // Midpoint rounds UP (AwayFromZero)
    [InlineData(100.004, 100.00)]   // Below midpoint rounds DOWN
    [InlineData(0.005,   0.01)]     // Paisa boundary — 0.5 paisa rounds to 1 paisa
    [InlineData(0.004,   0.00)]     // 0.4 paisa rounds to 0
    public void Money_Rounding_IsAlwaysAwayFromZero(decimal input, decimal expected)
        => Money.InrOf(input).Amount.Should().Be(expected);

    [Fact]
    public void Money_NegativeAmount_ThrowsDomainException()
        => FluentActions.Invoking(() => Money.InrOf(-1m))
                        .Should().Throw<DomainException>();

    [Fact]
    public void Money_CurrencyMismatch_ThrowsDomainException()
    {
        var inr = Money.InrOf(100m);
        var usd = Money.Of(100m, "USD");
        FluentActions.Invoking(() => inr + usd)
                     .Should().Throw<DomainException>();
    }
}
```

### 10.3 E2E Keyboard Navigation Tests

```typescript
// Playwright — keyboard-only. NEVER use click(), mouse.move(), etc.
test('accountant creates invoice using keyboard only', async ({ page }) => {
  // Navigate to invoices — keyboard only
  await page.keyboard.press('F10');           // Open main menu
  await page.keyboard.press('ArrowDown');     // Navigate to Invoices
  await page.keyboard.press('Enter');

  // Create new invoice
  await page.keyboard.press('Control+n');     // New invoice

  // Fill customer field
  await page.keyboard.press('Tab');
  await page.keyboard.type('ABC Traders');
  await page.keyboard.press('F4');            // Open lookup
  await page.keyboard.press('ArrowDown');     // Select first result
  await page.keyboard.press('Enter');

  // Fill date
  await page.keyboard.press('Tab');
  await page.keyboard.type('01/04/2025');

  // Add line item in grid
  await page.keyboard.press('Tab');
  await page.keyboard.type('Consulting Services');
  await page.keyboard.press('Tab');
  await page.keyboard.type('10000');          // Amount — no mouse needed

  // Post the invoice
  await page.keyboard.press('F9');            // Post / Save

  // Verify focus moved correctly — never lost
  const focused = await page.evaluate(
    () => document.activeElement?.getAttribute('data-testid')
  );
  expect(focused).toBe('invoice-saved-confirmation');

  // Verify total calculated correctly (₹10,000 + 18% GST = ₹11,800)
  const total = await page.locator('[data-testid=invoice-total]').innerText();
  expect(total).toBe('₹11,800.00');
});
```

### 10.4 Financial Calculation Test Checklist

Every financial calculation function must have tests for these cases:

**Money / Decimal edge cases:**
- [ ] `0.005` rounds to `0.01` (paisa boundary — midpoint)
- [ ] `0.004` rounds to `0.00` (below midpoint)
- [ ] `100 * 0.185 = 18.50` exactly (not `18.499...`)
- [ ] Large amounts: `99,99,999.99` (max realistic Indian invoice)
- [ ] Negative guard: cannot create `Money` with negative amount
- [ ] Currency mismatch throws `DomainException`
- [ ] Subtraction to zero: `100.00 - 100.00 = 0.00`
- [ ] Accumulation of many small amounts (float drift test)

**GST / Tax edge cases:**
- [ ] 18% on round amount: `1000 → 180.00`
- [ ] 18% on `1001 → 180.18`
- [ ] All slabs: 0%, 5%, 12%, 18%, 28%
- [ ] CGST + SGST = IGST total (intra vs inter state)
- [ ] GST on zero amount = zero
- [ ] Discount applied BEFORE tax (not after)
- [ ] Reverse calculation: gross with GST → base amount

---

## 11. Auto-Update System

### 11.1 Update Behavior

| Rule | Detail |
|------|--------|
| Check frequency | Every 4 hours background. Also on startup. Never blocks UI. |
| Download | Delta updates only — only changed files downloaded. |
| Notification | Subtle status bar banner: `"Update ready — apply at end of session (Alt+U)"` — never a popup. |
| Apply timing | **Never during an open entry form.** Waits for user to be on list/dashboard screen. |
| Rollback | Previous version kept 7 days. `Ctrl+Shift+R` opens rollback dialog (SuperAdmin only). |
| Changelog | Update notification includes "What's new" summary in user's current locale. |
| DB migrations | Run automatically after update. Must be backward-compatible. |
| Offline | Fully functional offline. Update check silently skipped. |

### 11.2 Migration Safety Rules

```csharp
// ✅ SAFE: Add nullable column with default
migrationBuilder.AddColumn<string>(
    "GstNumber", "Companies",
    nullable: true, defaultValue: null
);

// ✅ SAFE: Add new table
migrationBuilder.CreateTable("AuditLogs", /* ... */);

// ✅ SAFE: Rename via 3-release cycle
// Release N  : Add new column, copy data, keep old column
// Release N+1: Code uses new column, old column exists but unused
// Release N+2: Drop old column in a dedicated migration

// ❌ NEVER: Drop a column in a single migration (data loss risk)
migrationBuilder.DropColumn("OldField", "Invoices"); // BANNED

// ❌ NEVER: Change column type that could lose precision
// ❌ NEVER: Make nullable column non-nullable without a DEFAULT
// ❌ NEVER: Write a migration that cannot be rolled back
```

---

## 12. Security, Auth & Error Handling

### 12.1 Roles

| Role | Permissions |
|------|-------------|
| SuperAdmin | Full access: user management, company creation, audit logs, rollback, system config |
| Accountant | Create/edit invoices, post ledger entries, run all reports, export data |
| Auditor | Read-only: all financial data and audit trail. Cannot modify anything. |
| Manager | Approve/reject entries, view all reports. Cannot create or post directly. |
| Client | View own invoices, receipts, and statements only. |

> **Rule:** Every API endpoint must declare an explicit authorization policy. No anonymous endpoints except `/auth/login`, `/auth/refresh`, and `/health`.

### 12.2 JWT Configuration

| Setting | Value | Reason |
|---------|-------|--------|
| Access token expiry | 15 minutes | Limits damage from token theft |
| Refresh token expiry | 7 days | Stored in DB with revocation flag |
| Claims | `sub`, `email`, `roles[]`, `tenantId`, `companyId`, `jti` | Full context per request |
| Secret key | 256-bit minimum, environment variable only | Never in `appsettings.json` |
| Refresh rotation | New refresh token on every refresh | Old token immediately invalidated |

### 12.3 Exception Hierarchy

```csharp
// Base
public class AppException(string errorCode, string message) : Exception(message);

// Domain rule violation — HTTP 422
public class DomainException(string message)
    : AppException("DOMAIN_RULE", message);

// Entity not found — HTTP 404. Always include entity type + ID.
public class NotFoundException(string entityType, object id)
    : AppException("NOT_FOUND", $"{entityType} '{id}' not found.");

// Input validation — HTTP 400. Returns field-level error list.
public class ValidationException(IEnumerable<ValidationError> errors)
    : AppException("VALIDATION", "One or more validation errors occurred.");

// Concurrent update conflict — HTTP 409
public class ConflictException(string message)
    : AppException("CONFLICT", message);

// Authorization failure — HTTP 403
public class ForbiddenException(string reason)
    : AppException("FORBIDDEN", reason);
```

### 12.4 Logging Rules

```csharp
// ✅ CORRECT — structured logging with named properties
Log.Information(
    "Invoice {InvoiceId} created by {UserId} for company {CompanyId}",
    invoice.Id, currentUser.UserId, currentUser.CompanyId
);

// ❌ WRONG — string interpolation defeats structured logging
Log.Information($"Invoice {invoice.Id} created");  // Can't query by InvoiceId

// NEVER LOG: passwords, PAN, Aadhaar, bank account numbers, card numbers
// Use masking: Log.Information("Payment by card ending {Last4}", card.Last4);
```

---

## 13. AI Prompt Context Block

> **Paste this block at the TOP of every AI coding session.**  
> It takes 10 seconds. It saves hours of inconsistency.

```
---LEDGERAPP AI CONTEXT v2.0---
PROJECT: LedgerApp — Accounting software for Indian businesses (multi-company)

STACK:
  Backend:  ASP.NET Core 8, C# 12, Clean Architecture (4 layers: Domain/Application/Infrastructure/API)
  Pattern:  CQRS + MediatR, DDD (rich domain model, aggregates, value objects, domain events)
  Database: SQL Server + EF Core 8, Fluent API config ONLY — zero DataAnnotations on domain entities
  State:    Fluxor (Blazor) or Zustand (React) — single source of truth, no component-owned server data
  Realtime: SignalR — AccountingHub broadcasts EntityUpdated to all company members on every state change
  Auth:     JWT (15min access / 7d refresh + revocation), role-based policies, CompanyId isolation
  Logging:  Serilog structured, EF Core SaveChanges interceptor (all writes auto-captured to AuditLog)
  i18n:     en-IN (default), hi-IN, pa-IN — ZERO hardcoded UI strings — key format: Feature.Section.Key
  Testing:  xUnit + NSubstitute + FluentAssertions + TestContainers + Playwright (keyboard-only E2E)

CRITICAL RULES:
  MONEY:       Always use Money value object. decimal(18,2) in DB via Fluent API.
               MidpointRounding.AwayFromZero always. NEVER float/double for any financial value.
               GST rates in GstSlabs constants — never inline 0.18m.

  KEYBOARD:    ALL UI must be fully operable by keyboard only.
               Focus engine: Up/Down/Left/Right spatial navigation (Apple TV model).
               Dashed focus border: outline: 2px dashed var(--focus-ring-color); outline-offset: 3px
               Global shortcuts: Ctrl+N=New, Ctrl+S=Save, F9=Post, F2=Edit, F3=Search,
                                 F4=Lookup, F5=Refresh, F12=Print/Export,
                                 Ctrl+E=Export, Ctrl+I=Import, Ctrl+W=Switch company
               Always show shortcut in button label: "Save (F9)", "New (Ctrl+N)"

  STATE:       No component owns server data — all from central store.
               SignalR broadcasts changes → store updated → all views re-render automatically.
               One store slice per domain. Verb-noun actions: addInvoice, not setInvoices.

  MULTI-CO:    Every entity has CompanyId. All queries scoped via CompanyScopedRepository<T>.
               EF Core global query filter on all company entities as safety net.
               Export: .ledger (JSON+gzip+checksum). Import: schema check → dry-run → commit.
               Ctrl+W opens company switcher — no logout required.

  COLORS:      CSS variables ONLY — never hardcode hex in components.
               Teal=paid/approved/credit, Amber=pending/draft, Red=error/overdue/debit, Green=income.
               Light: --bg-base #F8FAFC, Dark: --bg-base #0F172A (pastel, not harsh).

  FONTS:       JetBrains Mono for ALL financial figures and account codes.
               Inter or Nunito for UI text. Minimum 15px — senior user audience.

  NAMING:      PascalCase classes, camelCase vars, _camelCase private fields.
               isX/hasX/canX booleans. Async suffix on all async methods.

  FILES:       200 lines max per file. 20 lines max per method. 1 class per file.
               XML doc comments on ALL public members. Explain WHY not WHAT.
               Accounting/tax rules must cite applicable law in comments.

  STRINGS:     All user-visible strings use i18n resource keys. Key format: Feature.Section.Key.
               Resources: en-IN.resx, hi-IN.resx, pa-IN.resx.

  UPDATES:     Auto-update downloads silently in background. NEVER interrupt mid-entry.
               DB migrations: always backward-compatible — no destructive column drops.
               3-release cycle for renames.

  TESTING:     Unit tests for all handlers (decimal edge cases mandatory).
               E2E tests use Playwright with keyboard-only (zero mouse events).

FOLDER PATTERN: src/LedgerApp.{Domain|Application|Infrastructure|API}/Features/{FeatureName}/
---END CONTEXT---
```

---

## 14. Master Scaffold Prompt

> **Use this to start the project from zero.** Open a new Claude chat. Paste the AI Context Block from Section 13 first. Then paste the prompt below.

```
[Paste AI Context Block from Section 13 first]

You are a senior .NET architect and UX engineer. Scaffold the complete LedgerApp
solution from zero applying EVERY standard in the context block above.

━━━ SOLUTION PROJECTS ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Create LedgerApp.sln with 6 projects:
  1. LedgerApp.Domain
  2. LedgerApp.Application
  3. LedgerApp.Infrastructure
  4. LedgerApp.API  (ASP.NET Core 8 Minimal APIs)
  5. LedgerApp.Tests.Unit
  6. LedgerApp.Tests.Integration

━━━ DOMAIN LAYER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Entities (private setters, static Create() factory method, domain event collection list):
  Company, Invoice, LineItem, LedgerEntry, JournalLine, Account, Payment

Value Objects (immutable records, equality by value):
  Money(decimal Amount, string Currency)
    - MidpointRounding.AwayFromZero
    - Arithmetic operators with currency guard
    - Display(locale) method → Indian format
  AccountCode, TaxRate (Calculate(Money) method), FiscalYear

Domain Events (INotification records):
  InvoiceCreatedEvent, PaymentReceivedEvent, LedgerPostedEvent, CompanySwitchedEvent

Interfaces:
  IRepository<T>, IReadRepository<T>, IUnitOfWork
  ICompanyContext (CompanyId, CompanyName, FiscalYearStart, DefaultCurrency, DefaultLocale)
  IDomainEventDispatcher

Exceptions:
  DomainException, NotFoundException, ValidationException, ConflictException, ForbiddenException

Constants:
  GstSlabs (0%, 5%, 12%, 18%, 28%), MaxLineItems, SupportedLocales, ExportSchemaVersion

━━━ APPLICATION LAYER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CQRS features — one file per use case in Features/{Name}/:
  Companies:       CreateCompanyCommand, SwitchCompanyCommand, GetCompaniesQuery
  Invoices:        CreateInvoiceCommand, UpdateInvoiceCommand, PostInvoiceCommand,
                   DeleteInvoiceCommand, GetInvoiceByIdQuery, GetInvoicesPagedQuery
  LedgerEntries:   PostLedgerEntryCommand, GetLedgerReportQuery, GetTrialBalanceQuery
  Payments:        RecordPaymentCommand, GetPaymentsByInvoiceQuery
  DataPortability: ExportCompanyDataCommand, ImportCompanyDataCommand
  Reports:         GetBalanceSheetQuery, GetProfitLossQuery

FluentValidation validators for every Command.

Pipeline Behaviors (in order):
  1. LoggingBehavior     — request name, userId, companyId, elapsed ms
  2. ValidationBehavior  — runs all validators, aggregates field errors
  3. TransactionBehavior — DB transaction for commands, dispatch domain events after commit

All DTOs as C# records. Never expose domain entities outside Application.

━━━ INFRASTRUCTURE LAYER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
AppDbContext:
  - Fluent API only (zero DataAnnotations on domain entities)
  - Money as owned entity (two columns: Amount + Currency)
  - Global query filters for CompanyId on all company-scoped entities
  - SaveChangesInterceptor → auto-captures all writes to AuditLog table

Repositories: CompanyScopedRepository<T> base auto-applies CompanyId filter
UnitOfWork: SaveChangesAsync, BeginTransactionAsync, CommitAsync, RollbackAsync

Services:
  - JwtTokenService: GenerateAccessToken + GenerateRefreshToken + Revoke
  - DataExportService: serialize to .ledger (JSON+gzip), verify checksum on import
  - AutoUpdateService: background check, delta download, deferred apply
  - SignalR AccountingHub: BroadcastUpdate to company group after every command

Serilog:
  - Structured logging (no string interpolation)
  - Sensitive field masking (PAN, Aadhaar, bank account, card numbers)
  - X-Correlation-Id enricher

━━━ API LAYER ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Minimal APIs only — no controllers.
Endpoint groups:
  /api/auth, /api/companies, /api/invoices, /api/ledger,
  /api/payments, /api/reports, /api/data, /health

Middleware pipeline (in order):
  CorrelationId → ExceptionHandler → RequestLogging → RateLimiting

Global exception handler:
  - Maps all AppException subtypes to RFC 7807 ProblemDetails
  - Never leaks stack trace in production
  - Includes X-Correlation-Id in every error response

Authorization:
  - Every endpoint has explicit policy
  - Zero anonymous endpoints except /auth/* and /health

SignalR: /hubs/accounting
Swagger: JWT bearer auth, XML doc comments, example values

━━━ KEYBOARD NAVIGATION ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
KeyboardNavigationService:
  - Registers all FocusZones with their spatial neighbors
  - Routes Up/Down/Left/Right between zones
  - Maintains focus history stack (Escape pops it)
  - Handles all global shortcuts (Ctrl+N, F9, Ctrl+W, etc.)

FocusZone component:
  - Dashed focus indicator: outline 2px dashed var(--focus-ring-color)
  - Light theme: #1B3A5C, Dark theme: #60A5FA
  - Grid navigation: Arrow keys, F2=Edit, F9=Save, Enter=Confirm, Escape=Cancel

━━━ STATE MANAGEMENT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
If Blazor: Fluxor — one feature state slice per domain.
If React:  Zustand — one store per domain, devtools middleware, selector hooks.
SignalR client: on EntityUpdated → dispatch action → reducer → all views update.

━━━ OUTPUT FORMAT ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Full folder tree for all 6 projects
2. All .csproj files with correct NuGet PackageReference versions
3. Program.cs — complete DI registration and middleware pipeline
4. Full code for these files:
   - Money.cs (complete value object)
   - Invoice.cs (aggregate root with domain event collection)
   - IUnitOfWork.cs + EfUnitOfWork.cs
   - CreateInvoiceCommand.cs + CreateInvoiceCommandHandler.cs
   - AppDbContext.cs with all Fluent API configurations
   - KeyboardNavigationService.cs + FocusZone component
   - ExportCompanyDataCommand.cs + Handler
   - AccountingHub.cs (SignalR)
   - GlobalExceptionMiddleware.cs
   - LoggingBehavior.cs + ValidationBehavior.cs + TransactionBehavior.cs
   - CreateInvoiceCommandHandlerTests.cs (with decimal/paisa edge cases)
   - InvoiceKeyboardNavigationTests.cs (Playwright, zero mouse events)
5. /docs/DOMAIN.md with accounting glossary for developers
6. /docs/CHANGELOG.md template
7. "Onboarding in 5 steps" section for a developer joining the project
```

---

## 15. Quick Reference Card

> Print this page and stick it on your monitor.

### NEVER do this

- Use raw `decimal`/`float`/`double` for money — always `Money.InrOf()`
- Hardcode a color hex value — always `var(--css-variable)`
- Hardcode a UI string — always an i18n resource key
- Put business logic in a handler or component — delegate to domain entity
- Call `AppDbContext` directly in a handler — inject `IUnitOfWork`
- Use `outline: none` without a replacement focus indicator
- Create a component or file over 200 lines
- Put more than one class in a file
- Write a DB migration that drops a column
- Apply an update while the user has an entry form open
- Skip the AI Context Block when starting a new AI coding session

### ALWAYS do this

- Use `Money.InrOf()` for all rupee amounts
- Test decimal edge cases: `0.005`, `0.004`, paisa boundary
- Paste Section 13 AI Context Block at the top of every AI chat
- Add XML doc to every public class, method, and interface
- Scope ALL queries to `CompanyId` via `ICompanyContext`
- Test every new screen keyboard-only — zero mouse events
- Use dashed focus border: `outline: 2px dashed var(--focus-ring-color)`
- Show keyboard shortcut in button labels: `Save (F9)`, `New (Ctrl+N)`
- Update `CHANGELOG.md` in the same PR as the code change
- Broadcast state changes via SignalR — no manual refresh
- Update this document when any convention changes

---

## Document Maintenance

> This is a **living document.**  
> Update it in the **same PR** as any code change that introduces or changes a convention.  
> When onboarding a new developer — hand them this document first.  
> When starting an AI coding session — paste Section 13 first.

| Version | Date | Changed |
|---------|------|---------|
| 2.0 | 2025-05-06 | Added keyboard navigation, reactive state, multi-company, export/import, auto-update, full testing protocol |
| 1.0 | 2025-05-01 | Initial standards document |
