# TanStack Vue Table retained for DataTable

**Status:** Accepted (amendment to shadcn-only table decision)

**Context:** The JDPL alignment plan called for shadcn-vue `Table` primitives everywhere. Several admin/list pages need sortable, filterable columns with server-side pagination.

**Decision:** Keep `@tanstack/vue-table` as a **headless** layer. The shared `DataTable` component composes TanStack column definitions and state with shadcn-vue `Table`, `TableHeader`, `TableBody`, `TableRow`, and `TableCell` for markup and styling.

**Rationale:** Reimplementing sorting, filtering, and column visibility on raw shadcn tables would duplicate TanStack behaviour without meaningful UX gain. This is a pragmatic hybrid: design tokens and shadcn primitives for presentation; TanStack for table logic only.

**Consequences:** New list pages should use the existing `DataTable` pattern rather than ad-hoc table markup or a second table library.
