<script lang="ts">
  import type { PluginSDK, Goal, Account, AccountAllocation } from "./types";

  // Props from plugin SDK
  const { sdk }: { sdk: PluginSDK } = $props();

  // ============================================================================
  // State
  // ============================================================================

  let loading = $state(true);
  let error = $state<string | null>(null);

  // Data
  let goals = $state<Goal[]>([]);
  let accounts = $state<Account[]>([]);
  let goalBalances = $state<Map<string, number>>(new Map());

  // UI state
  let showAddGoal = $state(false);
  let editingGoal = $state<Goal | null>(null);
  let showCompleted = $state(false);

  // Form state for new goal
  interface FormAllocation {
    account_id: string;
    allocation_type: "percentage" | "fixed";
    allocation_value: number;
  }

  let newGoalName = $state("");
  let newGoalTargetAmount = $state(0);
  let newGoalTargetDate = $state<string | null>(null);
  let newGoalIcon = $state("🎯");
  let newGoalColor = $state("#3b82f6");
  let newGoalAllocations = $state<FormAllocation[]>([]);

  // Preset goal templates
  const goalTemplates = [
    { name: "Emergency Fund", icon: "🛡️", color: "#ef4444", months: 6 },
    { name: "House Down Payment", icon: "🏠", color: "#22c55e", amount: 50000 },
    { name: "Vacation", icon: "✈️", color: "#8b5cf6", amount: 5000 },
    { name: "New Car", icon: "🚗", color: "#f59e0b", amount: 25000 },
    { name: "Wedding", icon: "💒", color: "#ec4899", amount: 20000 },
    { name: "Education", icon: "🎓", color: "#06b6d4", amount: 10000 },
  ];

  // ============================================================================
  // Initialization
  // ============================================================================

  async function initialize() {
    loading = true;
    error = null;
    try {
      await ensureTablesExist();
      await Promise.all([loadAccounts(), loadGoals()]);
      await calculateBalances();
    } catch (e) {
      error = e instanceof Error ? e.message : "Failed to initialize";
      console.error("Goals init error:", e);
    } finally {
      loading = false;
    }
  }

  async function ensureTablesExist() {
    // Drop old table if it has the old schema (account_ids instead of allocations)
    // This is a dev-time migration - in production you'd do proper migrations
    try {
      const cols = await sdk.query<unknown[]>(`
        SELECT column_name FROM information_schema.columns
        WHERE table_name = 'sys_plugin_goals' AND column_name = 'account_ids'
      `);
      if (cols.length > 0) {
        await sdk.execute(`DROP TABLE IF EXISTS sys_plugin_goals`);
      }
    } catch {
      // Table doesn't exist yet, that's fine
    }

    await sdk.execute(`
      CREATE TABLE IF NOT EXISTS sys_plugin_goals (
        id VARCHAR PRIMARY KEY,
        name VARCHAR NOT NULL,
        target_amount DECIMAL(15,2) NOT NULL,
        target_date DATE,
        allocations JSON,
        starting_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
        icon VARCHAR NOT NULL DEFAULT '🎯',
        color VARCHAR NOT NULL DEFAULT '#3b82f6',
        active BOOLEAN NOT NULL DEFAULT TRUE,
        completed BOOLEAN NOT NULL DEFAULT FALSE,
        completed_at TIMESTAMP,
        created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
      )
    `);
  }

  async function loadAccounts() {
    // Get accounts with latest balance from snapshots
    // Uses a subquery to get the most recent snapshot per account
    const result = await sdk.query<unknown[]>(`
      SELECT
        a.account_id,
        COALESCE(a.nickname, a.name) as name,
        COALESCE(latest.balance, a.balance, 0) as balance,
        a.account_type
      FROM accounts a
      LEFT JOIN (
        SELECT account_id, balance
        FROM sys_balance_snapshots s1
        WHERE snapshot_time = (
          SELECT MAX(snapshot_time)
          FROM sys_balance_snapshots s2
          WHERE s2.account_id = s1.account_id
        )
      ) latest ON a.account_id = latest.account_id
      ORDER BY name
    `);

    accounts = result.map((row) => ({
      account_id: row[0] as string,
      name: row[1] as string,
      balance: Number(row[2]) || 0,
      account_type: row[3] as string | null,
    }));
  }

  async function loadGoals() {
    const result = await sdk.query<unknown[]>(`
      SELECT id, name, target_amount, target_date, allocations, starting_balance,
             icon, color, active, completed, completed_at, created_at, updated_at
      FROM sys_plugin_goals
      ORDER BY completed ASC, created_at DESC
    `);

    goals = result.map((row) => ({
      id: row[0] as string,
      name: row[1] as string,
      target_amount: Number(row[2]) || 0,
      target_date: row[3] as string | null,
      allocations: row[4] ? JSON.parse(row[4] as string) : null,
      starting_balance: Number(row[5]) || 0,
      icon: row[6] as string,
      color: row[7] as string,
      active: Boolean(row[8]),
      completed: Boolean(row[9]),
      completed_at: row[10] as string | null,
      created_at: row[11] as string,
      updated_at: row[12] as string,
    }));
  }

  async function calculateBalances() {
    const balances = new Map<string, number>();

    for (const goal of goals) {
      if (goal.allocations && goal.allocations.length > 0) {
        // Sum allocated amounts from linked accounts
        let total = 0;
        for (const alloc of goal.allocations) {
          const account = accounts.find((a) => a.account_id === alloc.account_id);
          if (account) {
            if (alloc.allocation_type === "percentage") {
              // Percentage of account balance
              total += (account.balance * alloc.allocation_value) / 100;
            } else {
              // Fixed amount (capped at account balance)
              total += Math.min(alloc.allocation_value, account.balance);
            }
          }
        }
        balances.set(goal.id, total);
      } else {
        // No linked accounts - use starting balance (manual tracking)
        balances.set(goal.id, goal.starting_balance);
      }
    }

    goalBalances = balances;
  }

  // ============================================================================
  // Goal CRUD
  // ============================================================================

  async function createGoal() {
    if (!newGoalName || !newGoalTargetAmount) {
      sdk.toast.error("Name and target amount are required");
      return;
    }

    const id = crypto.randomUUID();
    const allocationsJson = newGoalAllocations.length > 0
      ? JSON.stringify(newGoalAllocations)
      : null;

    // Calculate starting balance from allocations
    let startingBalance = 0;
    for (const alloc of newGoalAllocations) {
      const account = accounts.find((a) => a.account_id === alloc.account_id);
      if (account) {
        if (alloc.allocation_type === "percentage") {
          startingBalance += (account.balance * alloc.allocation_value) / 100;
        } else {
          startingBalance += Math.min(alloc.allocation_value, account.balance);
        }
      }
    }

    await sdk.execute(`
      INSERT INTO sys_plugin_goals
        (id, name, target_amount, target_date, allocations, starting_balance, icon, color)
      VALUES (
        '${id}',
        '${escapeSql(newGoalName)}',
        ${newGoalTargetAmount},
        ${newGoalTargetDate ? `'${newGoalTargetDate}'` : "NULL"},
        ${allocationsJson ? `'${escapeSql(JSON.stringify(newGoalAllocations))}'` : "NULL"},
        ${startingBalance},
        '${newGoalIcon || "🎯"}',
        '${newGoalColor || "#3b82f6"}'
      )
    `);

    sdk.toast.success(`Goal "${newGoalName}" created!`);
    showAddGoal = false;
    resetForm();
    await loadGoals();
    await calculateBalances();
  }

  async function updateGoal(goal: Goal) {
    const allocationsJson = goal.allocations
      ? JSON.stringify(goal.allocations)
      : null;

    await sdk.execute(`
      UPDATE sys_plugin_goals SET
        name = '${escapeSql(goal.name)}',
        target_amount = ${goal.target_amount},
        target_date = ${goal.target_date ? `'${goal.target_date}'` : "NULL"},
        allocations = ${allocationsJson ? `'${escapeSql(allocationsJson)}'` : "NULL"},
        icon = '${goal.icon}',
        color = '${goal.color}',
        updated_at = CURRENT_TIMESTAMP
      WHERE id = '${goal.id}'
    `);

    sdk.toast.success("Goal updated");
    editingGoal = null;
    await loadGoals();
    await calculateBalances();
  }

  async function deleteGoal(id: string) {
    await sdk.execute(`DELETE FROM sys_plugin_goals WHERE id = '${id}'`);
    sdk.toast.success("Goal deleted");
    await loadGoals();
  }

  async function markComplete(goal: Goal) {
    await sdk.execute(`
      UPDATE sys_plugin_goals SET
        completed = TRUE,
        completed_at = CURRENT_TIMESTAMP,
        updated_at = CURRENT_TIMESTAMP
      WHERE id = '${goal.id}'
    `);
    sdk.toast.success(`Congratulations! Goal "${goal.name}" completed! 🎉`);
    await loadGoals();
  }

  async function reopenGoal(goal: Goal) {
    await sdk.execute(`
      UPDATE sys_plugin_goals SET
        completed = FALSE,
        completed_at = NULL,
        updated_at = CURRENT_TIMESTAMP
      WHERE id = '${goal.id}'
    `);
    await loadGoals();
  }

  function resetForm() {
    newGoalName = "";
    newGoalTargetAmount = 0;
    newGoalTargetDate = null;
    newGoalIcon = "🎯";
    newGoalColor = "#3b82f6";
    newGoalAllocations = [];
  }

  function applyTemplate(template: (typeof goalTemplates)[0]) {
    newGoalName = template.name;
    newGoalIcon = template.icon;
    newGoalColor = template.color;
    if ("amount" in template) {
      newGoalTargetAmount = template.amount;
    }
    // For emergency fund, calculate from monthly expenses
    if ("months" in template && template.months) {
      // This would need expense calculation - for now use placeholder
      newGoalTargetAmount = 10000; // Default, user can adjust
    }
  }

  function addAllocation() {
    if (accounts.length === 0) return;
    // Find first account not already allocated
    const usedIds = new Set(newGoalAllocations.map(a => a.account_id));
    const available = accounts.find(a => !usedIds.has(a.account_id));
    if (!available) {
      sdk.toast.warning("All accounts already allocated");
      return;
    }
    newGoalAllocations = [...newGoalAllocations, {
      account_id: available.account_id,
      allocation_type: "percentage" as const,
      allocation_value: 100,
    }];
  }

  function removeAllocation(index: number) {
    newGoalAllocations = newGoalAllocations.filter((_, i) => i !== index);
  }

  function addEditAllocation(goal: Goal) {
    if (!goal.allocations) goal.allocations = [];
    const usedIds = new Set(goal.allocations.map(a => a.account_id));
    const available = accounts.find(a => !usedIds.has(a.account_id));
    if (!available) {
      sdk.toast.warning("All accounts already allocated");
      return;
    }
    goal.allocations = [...goal.allocations, {
      account_id: available.account_id,
      allocation_type: "percentage" as const,
      allocation_value: 100,
    }];
  }

  function removeEditAllocation(goal: Goal, index: number) {
    if (!goal.allocations) return;
    goal.allocations = goal.allocations.filter((_, i) => i !== index);
  }

  // ============================================================================
  // Helpers
  // ============================================================================

  function escapeSql(str: string): string {
    return str.replace(/'/g, "''");
  }

  function formatCurrency(amount: number): string {
    return new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: "USD",
      minimumFractionDigits: 0,
      maximumFractionDigits: 0,
    }).format(amount);
  }

  function formatDate(dateStr: string): string {
    return new Date(dateStr).toLocaleDateString("en-US", {
      month: "short",
      day: "numeric",
      year: "numeric",
    });
  }

  function getProgress(goal: Goal): number {
    const current = goalBalances.get(goal.id) || 0;
    const saved = current - goal.starting_balance;
    const needed = goal.target_amount - goal.starting_balance;
    if (needed <= 0) return 100;
    return Math.min(100, Math.max(0, (saved / needed) * 100));
  }

  function getCurrentAmount(goal: Goal): number {
    return goalBalances.get(goal.id) || goal.starting_balance;
  }

  function getRemainingAmount(goal: Goal): number {
    const current = getCurrentAmount(goal);
    return Math.max(0, goal.target_amount - current);
  }

  function getDaysRemaining(goal: Goal): number | null {
    if (!goal.target_date) return null;
    const target = new Date(goal.target_date);
    const today = new Date();
    const diff = target.getTime() - today.getTime();
    return Math.ceil(diff / (1000 * 60 * 60 * 24));
  }

  function getMonthlyNeeded(goal: Goal): number | null {
    const days = getDaysRemaining(goal);
    if (days === null || days <= 0) return null;
    const remaining = getRemainingAmount(goal);
    const months = days / 30;
    return remaining / months;
  }

  function isOnTrack(goal: Goal): boolean | null {
    if (!goal.target_date) return null;
    const days = getDaysRemaining(goal);
    if (days === null) return null;

    const totalDays =
      (new Date(goal.target_date).getTime() -
        new Date(goal.created_at).getTime()) /
      (1000 * 60 * 60 * 24);
    const elapsed = totalDays - days;
    const expectedProgress = (elapsed / totalDays) * 100;
    const actualProgress = getProgress(goal);

    return actualProgress >= expectedProgress - 5; // 5% tolerance
  }

  // ============================================================================
  // Derived State
  // ============================================================================

  let activeGoals = $derived(goals.filter((g) => !g.completed));
  let completedGoals = $derived(goals.filter((g) => g.completed));
  let totalSaved = $derived(
    activeGoals.reduce((sum, g) => sum + (getCurrentAmount(g) - g.starting_balance), 0)
  );
  let totalTarget = $derived(
    activeGoals.reduce((sum, g) => sum + (g.target_amount - g.starting_balance), 0)
  );

  // ============================================================================
  // Lifecycle
  // ============================================================================

  $effect(() => {
    initialize();
  });

  $effect(() => {
    const unsubscribe = sdk.onDataRefresh(() => {
      initialize();
    });
    return unsubscribe;
  });
</script>

<div class="view">
  <header class="header">
    <h1>Savings Goals</h1>
    <button class="btn primary" onclick={() => (showAddGoal = true)}>
      + New Goal
    </button>
  </header>

  {#if loading}
    <div class="loading">
      <div class="spinner"></div>
      <p>Loading goals...</p>
    </div>
  {:else if error}
    <div class="error">
      <p>{error}</p>
      <button class="btn primary" onclick={() => initialize()}>Retry</button>
    </div>
  {:else}
    <!-- Summary -->
    {#if activeGoals.length > 0}
      <div class="summary">
        <div class="summary-item">
          <span class="summary-label">Total Progress</span>
          <span class="summary-value">
            {formatCurrency(totalSaved)} / {formatCurrency(totalTarget)}
          </span>
        </div>
        <div class="summary-bar">
          <div
            class="summary-fill"
            style="width: {totalTarget > 0 ? (totalSaved / totalTarget) * 100 : 0}%"
          ></div>
        </div>
      </div>
    {/if}

    <!-- Goals Grid -->
    <div class="content">
      {#if activeGoals.length === 0 && completedGoals.length === 0}
        <div class="empty">
          <p>No savings goals yet.</p>
          <p>Create a goal to start tracking your progress!</p>
          <button class="btn primary" onclick={() => (showAddGoal = true)}>
            Create Your First Goal
          </button>
        </div>
      {:else}
        <div class="goals-grid">
          {#each activeGoals as goal}
            {@const progress = getProgress(goal)}
            {@const current = getCurrentAmount(goal)}
            {@const remaining = getRemainingAmount(goal)}
            {@const daysLeft = getDaysRemaining(goal)}
            {@const monthlyNeeded = getMonthlyNeeded(goal)}
            {@const onTrack = isOnTrack(goal)}
            <div class="goal-card">
              <div class="goal-header">
                <span class="goal-icon">{goal.icon}</span>
                <div class="goal-title">
                  <h3>{goal.name}</h3>
                  {#if goal.target_date}
                    <span class="goal-date">
                      {#if daysLeft !== null && daysLeft > 0}
                        {daysLeft} days left
                      {:else if daysLeft !== null && daysLeft <= 0}
                        Past due
                      {/if}
                    </span>
                  {/if}
                </div>
                <div class="goal-actions">
                  <button
                    class="btn-icon"
                    onclick={() => (editingGoal = goal)}
                    title="Edit"
                  >
                    ✏️
                  </button>
                  <button
                    class="btn-icon"
                    onclick={() => deleteGoal(goal.id)}
                    title="Delete"
                  >
                    🗑️
                  </button>
                </div>
              </div>

              <div class="goal-progress">
                <div class="progress-bar">
                  <div
                    class="progress-fill"
                    style="width: {progress}%; background: {goal.color}"
                  ></div>
                </div>
                <div class="progress-labels">
                  <span>{formatCurrency(current)}</span>
                  <span class="progress-percent">{progress.toFixed(0)}%</span>
                  <span>{formatCurrency(goal.target_amount)}</span>
                </div>
              </div>

              <div class="goal-stats">
                <div class="stat">
                  <span class="stat-label">Remaining</span>
                  <span class="stat-value">{formatCurrency(remaining)}</span>
                </div>
                {#if monthlyNeeded !== null}
                  <div class="stat">
                    <span class="stat-label">Monthly needed</span>
                    <span class="stat-value">{formatCurrency(monthlyNeeded)}</span>
                  </div>
                {/if}
                {#if onTrack !== null}
                  <div class="stat">
                    <span class="stat-label">Status</span>
                    <span class="stat-value" class:on-track={onTrack} class:behind={!onTrack}>
                      {onTrack ? "On track" : "Behind"}
                    </span>
                  </div>
                {/if}
              </div>

              {#if goal.allocations && goal.allocations.length > 0}
                <div class="goal-accounts">
                  <span class="accounts-label">Tracking:</span>
                  {#each goal.allocations as alloc}
                    {@const account = accounts.find((a) => a.account_id === alloc.account_id)}
                    {#if account}
                      <span class="account-tag">
                        {account.name}
                        {#if alloc.allocation_type === "percentage"}
                          ({alloc.allocation_value}%)
                        {:else}
                          ({formatCurrency(alloc.allocation_value)})
                        {/if}
                      </span>
                    {/if}
                  {/each}
                </div>
              {/if}

              {#if progress >= 100}
                <button class="btn success full-width" onclick={() => markComplete(goal)}>
                  🎉 Mark Complete
                </button>
              {/if}
            </div>
          {/each}
        </div>

        <!-- Completed Goals -->
        {#if completedGoals.length > 0}
          <div class="completed-section">
            <button
              class="completed-toggle"
              onclick={() => (showCompleted = !showCompleted)}
            >
              {showCompleted ? "▼" : "▶"} Completed Goals ({completedGoals.length})
            </button>

            {#if showCompleted}
              <div class="completed-list">
                {#each completedGoals as goal}
                  <div class="completed-item">
                    <span class="goal-icon">{goal.icon}</span>
                    <span class="completed-name">{goal.name}</span>
                    <span class="completed-amount">{formatCurrency(goal.target_amount)}</span>
                    <span class="completed-date">
                      {goal.completed_at ? formatDate(goal.completed_at) : ""}
                    </span>
                    <button
                      class="btn text small"
                      onclick={() => reopenGoal(goal)}
                    >
                      Reopen
                    </button>
                    <button
                      class="btn text small danger"
                      onclick={() => deleteGoal(goal.id)}
                    >
                      Delete
                    </button>
                  </div>
                {/each}
              </div>
            {/if}
          </div>
        {/if}
      {/if}
    </div>
  {/if}

  <!-- Add Goal Modal -->
  {#if showAddGoal}
    <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
    <div class="modal-backdrop" onclick={() => (showAddGoal = false)}>
      <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
      <div class="modal" onclick={(e) => e.stopPropagation()}>
        <h3>Create New Goal</h3>

        <!-- Templates -->
        <div class="templates">
          <span class="templates-label">Quick start:</span>
          <div class="template-buttons">
            {#each goalTemplates as template}
              <button
                class="template-btn"
                onclick={() => applyTemplate(template)}
              >
                {template.icon} {template.name}
              </button>
            {/each}
          </div>
        </div>

        <div class="form">
          <div class="form-row">
            <label class="form-label icon-label">
              Icon
              <input
                type="text"
                bind:value={newGoalIcon}
                class="icon-input"
                maxlength="2"
              />
            </label>
            <label class="form-label color-label">
              Color
              <input type="color" bind:value={newGoalColor} class="color-input" />
            </label>
          </div>

          <label class="form-label">
            Goal Name
            <input
              type="text"
              bind:value={newGoalName}
              placeholder="e.g., Emergency Fund"
            />
          </label>

          <label class="form-label">
            Target Amount
            <input
              type="number"
              bind:value={newGoalTargetAmount}
              step="100"
              min="0"
              placeholder="10000"
            />
          </label>

          <label class="form-label">
            Target Date (optional)
            <input type="date" bind:value={newGoalTargetDate} />
          </label>

          <div class="allocations-section">
            <div class="allocations-header">
              <span class="form-label">Account Allocations</span>
              <button type="button" class="btn text small" onclick={addAllocation}>
                + Add Account
              </button>
            </div>
            {#if newGoalAllocations.length === 0}
              <p class="form-hint">No accounts linked. Goal will be tracked manually.</p>
            {:else}
              {#each newGoalAllocations as alloc, i}
                <div class="allocation-row">
                  <select bind:value={alloc.account_id} class="allocation-account">
                    {#each accounts as account}
                      <option value={account.account_id}>
                        {account.name} ({formatCurrency(account.balance)})
                      </option>
                    {/each}
                  </select>
                  <select bind:value={alloc.allocation_type} class="allocation-type">
                    <option value="percentage">%</option>
                    <option value="fixed">$</option>
                  </select>
                  <input
                    type="number"
                    bind:value={alloc.allocation_value}
                    class="allocation-value"
                    min="0"
                    max={alloc.allocation_type === "percentage" ? 100 : undefined}
                    step={alloc.allocation_type === "percentage" ? 1 : 100}
                  />
                  <button
                    type="button"
                    class="btn-icon danger"
                    onclick={() => removeAllocation(i)}
                  >
                    ✕
                  </button>
                </div>
              {/each}
            {/if}
          </div>
        </div>

        <div class="modal-actions">
          <button
            class="btn secondary"
            onclick={() => {
              showAddGoal = false;
              resetForm();
            }}
          >
            Cancel
          </button>
          <button class="btn primary" onclick={createGoal}>Create Goal</button>
        </div>
      </div>
    </div>
  {/if}

  <!-- Edit Goal Modal -->
  {#if editingGoal}
    <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
    <div class="modal-backdrop" onclick={() => (editingGoal = null)}>
      <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
      <div class="modal" onclick={(e) => e.stopPropagation()}>
        <h3>Edit Goal</h3>

        <div class="form">
          <div class="form-row">
            <label class="form-label icon-label">
              Icon
              <input
                type="text"
                bind:value={editingGoal.icon}
                class="icon-input"
                maxlength="2"
              />
            </label>
            <label class="form-label color-label">
              Color
              <input type="color" bind:value={editingGoal.color} class="color-input" />
            </label>
          </div>

          <label class="form-label">
            Goal Name
            <input type="text" bind:value={editingGoal.name} />
          </label>

          <label class="form-label">
            Target Amount
            <input
              type="number"
              bind:value={editingGoal.target_amount}
              step="100"
              min="0"
            />
          </label>

          <label class="form-label">
            Target Date (optional)
            <input type="date" bind:value={editingGoal.target_date} />
          </label>

          <div class="allocations-section">
            <div class="allocations-header">
              <span class="form-label">Account Allocations</span>
              <button type="button" class="btn text small" onclick={() => addEditAllocation(editingGoal!)}>
                + Add Account
              </button>
            </div>
            {#if !editingGoal.allocations || editingGoal.allocations.length === 0}
              <p class="form-hint">No accounts linked. Goal is tracked manually.</p>
            {:else}
              {#each editingGoal.allocations as alloc, i}
                <div class="allocation-row">
                  <select bind:value={alloc.account_id} class="allocation-account">
                    {#each accounts as account}
                      <option value={account.account_id}>
                        {account.name} ({formatCurrency(account.balance)})
                      </option>
                    {/each}
                  </select>
                  <select bind:value={alloc.allocation_type} class="allocation-type">
                    <option value="percentage">%</option>
                    <option value="fixed">$</option>
                  </select>
                  <input
                    type="number"
                    bind:value={alloc.allocation_value}
                    class="allocation-value"
                    min="0"
                    max={alloc.allocation_type === "percentage" ? 100 : undefined}
                    step={alloc.allocation_type === "percentage" ? 1 : 100}
                  />
                  <button
                    type="button"
                    class="btn-icon danger"
                    onclick={() => removeEditAllocation(editingGoal!, i)}
                  >
                    ✕
                  </button>
                </div>
              {/each}
            {/if}
          </div>
        </div>

        <div class="modal-actions">
          <button class="btn secondary" onclick={() => (editingGoal = null)}>
            Cancel
          </button>
          <button class="btn primary" onclick={() => updateGoal(editingGoal!)}>
            Save Changes
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  .view {
    height: 100%;
    display: flex;
    flex-direction: column;
    background: var(--bg-primary);
    color: var(--text-primary);
  }

  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: var(--spacing-md) var(--spacing-lg);
    background: var(--bg-secondary);
    border-bottom: 1px solid var(--border-primary);
  }

  .header h1 {
    font-size: 18px;
    font-weight: 600;
    margin: 0;
  }

  .loading,
  .error,
  .empty {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    flex: 1;
    color: var(--text-muted);
    gap: var(--spacing-md);
    padding: var(--spacing-xl);
    text-align: center;
  }

  .spinner {
    width: 24px;
    height: 24px;
    border: 2px solid var(--border-primary);
    border-top-color: var(--accent-primary);
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .summary {
    padding: var(--spacing-md) var(--spacing-lg);
    background: var(--bg-secondary);
    border-bottom: 1px solid var(--border-primary);
  }

  .summary-item {
    display: flex;
    justify-content: space-between;
    margin-bottom: var(--spacing-sm);
  }

  .summary-label {
    font-size: 12px;
    color: var(--text-muted);
    text-transform: uppercase;
  }

  .summary-value {
    font-size: 14px;
    font-weight: 600;
    font-family: var(--font-mono);
  }

  .summary-bar {
    height: 6px;
    background: var(--bg-tertiary);
    border-radius: 3px;
    overflow: hidden;
  }

  .summary-fill {
    height: 100%;
    background: var(--accent-primary);
    border-radius: 3px;
    transition: width 0.3s ease;
  }

  .content {
    flex: 1;
    overflow-y: auto;
    padding: var(--spacing-lg);
  }

  .goals-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: var(--spacing-lg);
  }

  .goal-card {
    background: var(--bg-secondary);
    border: 1px solid var(--border-primary);
    border-radius: 8px;
    padding: var(--spacing-md);
  }

  .goal-header {
    display: flex;
    align-items: flex-start;
    gap: var(--spacing-sm);
    margin-bottom: var(--spacing-md);
  }

  .goal-icon {
    font-size: 24px;
    line-height: 1;
  }

  .goal-title {
    flex: 1;
  }

  .goal-title h3 {
    margin: 0;
    font-size: 16px;
    font-weight: 600;
  }

  .goal-date {
    font-size: 12px;
    color: var(--text-muted);
  }

  .goal-actions {
    display: flex;
    gap: 4px;
  }

  .btn-icon {
    background: none;
    border: none;
    padding: 4px;
    cursor: pointer;
    opacity: 0.5;
    font-size: 14px;
    transition: opacity 0.15s;
  }

  .btn-icon:hover {
    opacity: 1;
  }

  .goal-progress {
    margin-bottom: var(--spacing-md);
  }

  .progress-bar {
    height: 8px;
    background: var(--bg-tertiary);
    border-radius: 4px;
    overflow: hidden;
    margin-bottom: var(--spacing-xs);
  }

  .progress-fill {
    height: 100%;
    border-radius: 4px;
    transition: width 0.3s ease;
  }

  .progress-labels {
    display: flex;
    justify-content: space-between;
    font-size: 12px;
    color: var(--text-muted);
    font-family: var(--font-mono);
  }

  .progress-percent {
    font-weight: 600;
    color: var(--text-primary);
  }

  .goal-stats {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
    gap: var(--spacing-sm);
    margin-bottom: var(--spacing-md);
  }

  .stat {
    text-align: center;
    padding: var(--spacing-sm);
    background: var(--bg-tertiary);
    border-radius: 4px;
  }

  .stat-label {
    display: block;
    font-size: 10px;
    color: var(--text-muted);
    text-transform: uppercase;
    margin-bottom: 2px;
  }

  .stat-value {
    font-size: 13px;
    font-weight: 600;
    font-family: var(--font-mono);
  }

  .stat-value.on-track {
    color: var(--accent-success);
  }

  .stat-value.behind {
    color: var(--accent-warning);
  }

  .goal-accounts {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
    align-items: center;
    margin-bottom: var(--spacing-md);
  }

  .accounts-label {
    font-size: 11px;
    color: var(--text-muted);
  }

  .account-tag {
    font-size: 11px;
    padding: 2px 6px;
    background: var(--bg-tertiary);
    border-radius: 4px;
    color: var(--text-secondary);
  }

  .completed-section {
    margin-top: var(--spacing-xl);
    border-top: 1px solid var(--border-primary);
    padding-top: var(--spacing-md);
  }

  .completed-toggle {
    background: none;
    border: none;
    color: var(--text-muted);
    cursor: pointer;
    font-size: 13px;
    padding: var(--spacing-sm) 0;
  }

  .completed-toggle:hover {
    color: var(--text-primary);
  }

  .completed-list {
    margin-top: var(--spacing-sm);
  }

  .completed-item {
    display: flex;
    align-items: center;
    gap: var(--spacing-sm);
    padding: var(--spacing-sm);
    border-radius: 4px;
  }

  .completed-item:hover {
    background: var(--bg-secondary);
  }

  .completed-name {
    flex: 1;
    font-size: 13px;
  }

  .completed-amount {
    font-family: var(--font-mono);
    font-size: 13px;
    color: var(--text-muted);
  }

  .completed-date {
    font-size: 12px;
    color: var(--text-muted);
  }

  /* Buttons */
  .btn {
    padding: 8px 16px;
    border-radius: 4px;
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    border: none;
    transition: all 0.15s ease;
  }

  .btn.primary {
    background: var(--accent-primary);
    color: white;
  }

  .btn.primary:hover {
    opacity: 0.9;
  }

  .btn.secondary {
    background: var(--bg-tertiary);
    border: 1px solid var(--border-primary);
    color: var(--text-primary);
  }

  .btn.success {
    background: var(--accent-success);
    color: white;
  }

  .btn.text {
    background: none;
    padding: 4px 8px;
    color: var(--text-secondary);
  }

  .btn.text:hover {
    color: var(--text-primary);
  }

  .btn.text.danger:hover {
    color: var(--accent-danger);
  }

  .btn.small {
    font-size: 11px;
    padding: 4px 8px;
  }

  .btn.full-width {
    width: 100%;
  }

  /* Modal */
  .modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal {
    background: var(--bg-secondary);
    border: 1px solid var(--border-primary);
    border-radius: 8px;
    padding: var(--spacing-lg);
    width: 450px;
    max-width: 90vw;
    max-height: 90vh;
    overflow-y: auto;
  }

  .modal h3 {
    margin: 0 0 var(--spacing-md) 0;
    font-size: 16px;
    font-weight: 600;
  }

  .templates {
    margin-bottom: var(--spacing-lg);
  }

  .templates-label {
    display: block;
    font-size: 12px;
    color: var(--text-muted);
    margin-bottom: var(--spacing-sm);
  }

  .template-buttons {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }

  .template-btn {
    padding: 6px 10px;
    font-size: 12px;
    background: var(--bg-tertiary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    cursor: pointer;
    color: var(--text-primary);
    transition: all 0.15s;
  }

  .template-btn:hover {
    border-color: var(--accent-primary);
  }

  .form {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-md);
  }

  .form-row {
    display: flex;
    gap: var(--spacing-md);
  }

  .form-label {
    display: flex;
    flex-direction: column;
    gap: var(--spacing-xs);
    font-size: 13px;
    color: var(--text-secondary);
  }

  .form-label.icon-label {
    width: 80px;
  }

  .form-label.color-label {
    width: 80px;
  }

  .form-label input,
  .form-label select {
    padding: 8px;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    color: var(--text-primary);
    font-size: 14px;
  }

  .form-label input:focus,
  .form-label select:focus {
    outline: none;
    border-color: var(--accent-primary);
  }

  .icon-input {
    text-align: center;
    font-size: 20px !important;
  }

  .color-input {
    height: 38px;
    padding: 4px !important;
    cursor: pointer;
  }

  .account-select {
    min-height: 100px;
  }

  .form-hint {
    font-size: 11px;
    color: var(--text-muted);
  }

  .modal-actions {
    display: flex;
    justify-content: flex-end;
    gap: var(--spacing-sm);
    margin-top: var(--spacing-lg);
  }

  /* Allocations */
  .allocations-section {
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    padding: var(--spacing-md);
    background: var(--bg-tertiary);
  }

  .allocations-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: var(--spacing-sm);
  }

  .allocations-header .form-label {
    margin: 0;
    font-weight: 500;
  }

  .allocation-row {
    display: flex;
    gap: var(--spacing-sm);
    align-items: center;
    margin-bottom: var(--spacing-sm);
  }

  .allocation-row:last-child {
    margin-bottom: 0;
  }

  .allocation-account {
    flex: 1;
    padding: 6px 8px;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    color: var(--text-primary);
    font-size: 13px;
  }

  .allocation-type {
    width: 50px;
    padding: 6px 4px;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    color: var(--text-primary);
    font-size: 13px;
    text-align: center;
  }

  .allocation-value {
    width: 80px;
    padding: 6px 8px;
    background: var(--bg-primary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    color: var(--text-primary);
    font-size: 13px;
    font-family: var(--font-mono);
  }

  .btn-icon.danger:hover {
    color: var(--accent-danger);
  }
</style>
