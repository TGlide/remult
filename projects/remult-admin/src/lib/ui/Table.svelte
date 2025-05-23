<script lang="ts">
  import type {
    Repository,
    FindOptions,
    EntityFilter,
  } from '../../../../core/src/remult3/remult3'
  import type {
    EntityRelationToManyInfo,
    FieldUIInfo,
    RelationsToOneValues,
  } from '../../../../core/server/remult-admin'
  import { onDestroy } from 'svelte'
  import EditableRow from './EditableRow.svelte'
  import Filter from './Filter.svelte'
  import { writable, type Writable } from 'svelte/store'
  import LoadingSkeleton from './LoadingSkeleton.svelte'
  import { SSContext } from '../stores/SSContext.js'
  import { LSContext } from '../stores/LSContext.js'
  import Key from '../icons/Key.svelte'
  import ColumnType from '../icons/ColumnType.svelte'
  import { dialog } from './dialog/dialog.js'
  import ChevronLeft from '../icons/ChevronLeft.svelte'
  import ChevronRight from '../icons/ChevronRight.svelte'
  import Back from '../icons/Back.svelte'
  import { godStore } from '../../stores/GodStore.js'
  import Asc from '../icons/Asc.svelte'
  import Desc from '../icons/Desc.svelte'

  export let fields: FieldUIInfo[]
  export let relations: EntityRelationToManyInfo[]
  export let repo: Repository<any>
  export let parentRelation: Record<string, any> = {}
  export let color: string
  export let defaultOrderBy: EntityFilter<any>
  export let defaultNumberOfRows = 25

  let options: FindOptions<any>

  let errorMsg = ''

  // Reset to page 1 on key change
  $: options = repo.metadata.key && {
    limit: $LSContext.settings.numberOfRows,
    page: 1,
    orderBy: defaultOrderBy,
  }

  let filter: Writable<EntityFilter<any>> = writable({})
  let items: any[] | null = null
  let relationsToOneValues: RelationsToOneValues = {}

  $: $SSContext.forbiddenEntities.includes(repo.metadata.key) && (items = [])

  // resting when fields change
  $: items = fields && (items = null)
  $: relationsToOneValues = fields && (relationsToOneValues = {})
  $: $filter = fields && ($filter = {})

  let totalRows = -1
  let unSub: (() => void) | null = null

  let disableLiveQuery = false
  const updateDisableLiveQuery = (_disableLiveQuery: any) => {
    disableLiveQuery =
      _disableLiveQuery === undefined
        ? window.optionsFromServer?.disableLiveQuery ?? disableLiveQuery
        : _disableLiveQuery
  }
  $: updateDisableLiveQuery($LSContext.settings.disableLiveQuery)

  const reSub = async (currentFilter: EntityFilter<any>) => {
    $SSContext.forbiddenEntities = []

    if (unSub) {
      unSub()
    }

    const where = { $and: [currentFilter, { ...parentRelation }] }

    if (disableLiveQuery) {
      try {
        items = await repo.find({
          ...options,
          where,
        })
        await afterMainQuery(items, where)
        errorMsg = ''
      } catch (error) {
        errorMsg = JSON.stringify(error, null, 2)
      }
    } else {
      unSub = repo
        .liveQuery({
          ...options,
          where,
        })
        .subscribe({
          next: async (info) => {
            let tmpItems = items
            let special = false
            if (items !== null) {
              info.changes.forEach((c) => {
                if (c.type === 'add') {
                  special = true
                  tmpItems = [c.data.item, ...items]
                }
              })
            }

            if (!special) {
              tmpItems = info.applyChanges(items)
            }

            if (tmpItems && tmpItems.length > 0) {
              await afterMainQuery(tmpItems, where)
            }

            items = tmpItems
            errorMsg = ''
          },
          error(error) {
            errorMsg = JSON.stringify(error, null, 2)
          },
        })
    }
  }

  const afterMainQuery = async (items: any[], where: EntityFilter<any>) => {
    const promises = fields
      .filter((f) => f.relationToOne)
      .map((f) => $godStore.displayValueForEach(f, items))

    const results = await Promise.all(promises)
    results.forEach((r) => {
      relationsToOneValues = {
        ...relationsToOneValues,
        ...r,
      }
    })

    totalRows = await repo.count(where)
  }

  onDestroy(() => {
    unSub && unSub()
  })

  // trick to make sure reSub is called when repo changes
  $: repo && options && reSub($filter)

  $: from = ((options.page || 1) - 1) * options.limit + 1
  $: to = ((options.page || 1) - 1) * options.limit + (items?.length || 0)

  // Reset newRow when items change
  $: newRow = items && undefined

  const toggleOrderBy = (key: string) => {
    let dir = options.orderBy?.[key]
    if (dir === undefined) dir = 'desc'
    else if (dir === 'desc') dir = 'asc'
    else dir = undefined
    options = { ...options, orderBy: { [key]: dir } }
  }

  const getWidth = () => {
    const r = Math.random()

    if (r > 0.6) {
      return 120
    }
    if (r > 0.3) {
      return 100
    }
    return 70
  }

  const filterDialog = async (
    _fields: FieldUIInfo[],
    _filter: EntityFilter<any>,
  ) => {
    const res = await dialog.show<EntityFilter<any>>({
      config: { title: 'Filter', width: '600px' },
      component: Filter,
      props: { fields: _fields, filter: _filter },
    })
    if (res.success) {
      $filter = res.data
    }
  }
</script>

<div class="page-bar">
  <button
    class="icon-button"
    on:click={() => {
      const nav = document.querySelector('body')
      nav?.classList.toggle('hide-navigation')
    }}
  >
    <Back></Back>
  </button>

  <div class="page-bar__title title" style="--color: {color}">
    {$LSContext.settings.dispayCaption
      ? repo.metadata.caption
      : repo.metadata.key}
    {#if $SSContext.forbiddenEntities.includes(repo.metadata.key)}
      <span style="color: coral; font-size: smaller;">Forbidden</span>
    {/if}
  </div>
  <button on:click={() => filterDialog(fields, $filter)}> Filter</button>

  <span class="page-bar__results"
    >{from + ' - ' + to} of
    <b style="font-weight: 600;">{totalRows}</b></span
  >

  <button
    class="icon-button"
    disabled={(options.page || 1) === 1}
    on:click={() => {
      options = { ...options, page: (options.page || 2) - 1 }
    }}
  >
    <ChevronLeft></ChevronLeft>
  </button>

  <button
    class="icon-button"
    disabled={to >= totalRows}
    on:click={() => {
      options = { ...options, page: (options.page || 1) + 1 }
    }}
  >
    <ChevronRight></ChevronRight>
  </button>
</div>

<div class="table-container">
  {#if errorMsg}
    <pre class="error">{errorMsg.replaceAll(
        '\\n',
        `
`,
      )}</pre>
  {:else}
    <table>
      <thead>
        <tr>
          <td>
            {#if newRow === undefined}
              <button
                class="icon-button new-entry"
                on:click={() => {
                  newRow = repo.create({ ...parentRelation })
                }}
              >
                +
              </button>
            {:else}
              <button
                class="icon-button new-entry"
                on:click={() => {
                  newRow = undefined
                }}
              >
                -
              </button>
            {/if}
          </td>
          {#each fields as column}
            <th on:click={() => toggleOrderBy(column.key)}>
              <span class="th-span">
                {#if Object.keys(repo.metadata.options.id).includes(column.key)}
                  <Key></Key>
                {:else}
                  <span></span>
                {/if}
                <span class="flexItemCenter">
                  {$LSContext.settings.dispayCaption
                    ? column.caption
                    : column.key}
                  {#if options.orderBy?.[column.key] === 'asc'}
                    <Asc></Asc>
                  {:else if options.orderBy?.[column.key] === 'desc'}
                    <Desc></Desc>
                  {:else}
                    <span class="w-20"></span>
                  {/if}
                </span>
                <ColumnType
                  type={column.type}
                  isSelect={column.values && column.values.length > 0}
                ></ColumnType>
              </span>
            </th>
          {/each}
          <th class="action-tab">Actions</th>
        </tr>
      </thead>
      <tbody>
        {#if newRow}
          <EditableRow
            isNewRow
            rowId={undefined}
            row={newRow}
            {relationsToOneValues}
            columns={fields}
            {relations}
            saveAction={async (item) => {
              await repo.insert(item)
              newRow = undefined
              if (disableLiveQuery) {
                reSub($filter)
              }
            }}
            deleteAction={async () => {
              newRow = undefined
              if (disableLiveQuery) {
                reSub($filter)
              }
            }}
            cancelAction={async () => {
              newRow = undefined
              if (disableLiveQuery) {
                reSub($filter)
              }
            }}
          />
        {/if}
        {#if items}
          {#each items as row (repo.metadata.idMetadata.getId(row))}
            <EditableRow
              rowId={repo.metadata.idMetadata.getId(row)}
              {row}
              {relationsToOneValues}
              saveAction={async (item) => {
                await repo.update(row, item)
                if (disableLiveQuery) {
                  reSub($filter)
                }
              }}
              deleteAction={async () => {
                await repo.delete(row)
                if (disableLiveQuery) {
                  reSub($filter)
                }
              }}
              columns={fields}
              {relations}
            />
          {/each}
        {:else}
          {#each Array.from( { length: defaultNumberOfRows }, ).map((_, i) => i) as i}
            <tr>
              <td></td>
              {#each fields as column}
                <td
                  class="loading-skeleton"
                  on:click={() => toggleOrderBy(column.key)}
                >
                  <LoadingSkeleton width={getWidth()} />
                </td>
              {/each}
            </tr>
          {/each}
        {/if}
      </tbody>
    </table>
  {/if}
</div>

<style>
  .error {
    color: red;
    margin: 1rem;
    padding: 1rem;
  }

  .w-20 {
    width: 20px;
  }

  .flexItemCenter {
    display: flex;
    align-items: center;
  }
  th {
    cursor: pointer;
  }

  .th-span {
    display: flex;
    gap: 0.3rem;
    align-items: center;
    justify-content: space-between;
  }

  .title {
    border-left: 2px solid hsla(var(--color), 70%, 50%, 1);
  }

  .loading-skeleton {
    padding: 0 0.5rem;
  }

  button:disabled {
    opacity: 0.2;
    cursor: not-allowed;
  }
</style>
