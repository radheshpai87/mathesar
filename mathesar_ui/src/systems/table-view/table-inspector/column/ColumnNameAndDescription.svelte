<!--
  @component

  Enables inline editing of column name and description with a consolidated save action.
-->
<script lang="ts">
  import { _ } from 'svelte-i18n';

  import {
    CancelOrProceedButtonPair,
    LabeledInput,
    TextArea,
    TextInput,
  } from '@mathesar-component-library';
  import type {
    ColumnsDataStore,
    ProcessedColumn,
  } from '@mathesar/stores/table-data';
  import { toast } from '@mathesar/stores/toast';
  import { getErrorMessage } from '@mathesar/utils/errors';

  export let column: ProcessedColumn;
  export let columnsDataStore: ColumnsDataStore;
  export let currentRoleOwnsTable: boolean;

  $: ({ columns } = columnsDataStore);

  let isEditing = false;
  let name = '';
  let description = '';
  let isSubmitting = false;

  $: if (!isEditing) {
    name = column.column.name;
    description = column.column.description ?? '';
  }

  function getValidationErrors(newName: string): string[] {
    if (newName === column.column.name) {
      return [];
    }
    if (!newName) {
      return [$_('column_name_cannot_be_empty')];
    }
    const columnNames = $columns.map((c) => c.name);
    if (columnNames.includes(newName)) {
      return [$_('column_name_already_exists')];
    }
    return [];
  }

  $: validationErrors = getValidationErrors(name);
  $: hasChanges =
    name !== column.column.name ||
    description !== (column.column.description ?? '');
  $: canSave = validationErrors.length === 0 && hasChanges;

  function startEditing() {
    if (!currentRoleOwnsTable) return;
    name = column.column.name;
    description = column.column.description ?? '';
    isEditing = true;
  }

  function handleCancel() {
    name = column.column.name;
    description = column.column.description ?? '';
    isEditing = false;
  }

  async function handleSave() {
    if (!canSave) return;

    isSubmitting = true;
    try {
      await columnsDataStore.updateNameAndDescription(
        column.column.id,
        name,
        description || null,
      );
      isEditing = false;
    } catch (error) {
      toast.error(
        `${$_('unable_to_update_column')} ${getErrorMessage(error)}`,
      );
    } finally {
      isSubmitting = false;
    }
  }
</script>

<div class="column-properties">
  {#if !isEditing}
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="column-property" on:click={startEditing}>
      <span class="label">{$_('column_name')}</span>
      <TextInput value={column.column.name} disabled={!currentRoleOwnsTable} />
    </div>

    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div class="column-property" on:click={startEditing}>
      <span class="label">{$_('column_description')}</span>
      <TextArea value={column.column.description ?? ''} disabled={!currentRoleOwnsTable} />
    </div>
  {:else}
    <div class="editing-container">
      <LabeledInput label={$_('column_name')} layout="stacked">
        <TextInput bind:value={name} disabled={isSubmitting} autofocus />
        {#if validationErrors.length}
          {#each validationErrors as error}
            <span class="error">{error}</span>
          {/each}
        {/if}
      </LabeledInput>

      <LabeledInput label={$_('column_description')} layout="stacked">
        <TextArea bind:value={description} disabled={isSubmitting} />
      </LabeledInput>

      <CancelOrProceedButtonPair
        onProceed={handleSave}
        onCancel={handleCancel}
        isProcessing={isSubmitting}
        canProceed={canSave}
        proceedButton={{ label: $_('save') }}
        size="small"
      />
    </div>
  {/if}
</div>

<style lang="scss">
  .column-properties {
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .column-property {
    display: flex;
    flex-direction: column;
    cursor: pointer;

    .label {
      color: var(--color-fg-label);
    }

    > :global(* + *) {
      margin-top: 0.25rem;
    }
  }

  .editing-container {
    display: flex;
    flex-direction: column;

    > :global(* + *) {
      margin-top: 0.5rem;
    }
  }

  .error {
    color: var(--color-fg-danger);
    font-size: var(--sm2);
  }
</style>
