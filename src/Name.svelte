<script lang="ts">
  import { onMount, getContext } from "svelte";

  export let group: any;

  export let selected: any;
  export let selected_init: any;
  let selected_name = "";
  let selected_idx: number;

  let to: string | number | NodeJS.Timeout;
  let names = [];
  let url = getContext("LUOXU_URL");
  let input: HTMLInputElement;
  let ul: HTMLUListElement;
  let should_hide = false;

  let abort = new AbortController();

  onMount(() => {
    const rect = input.getBoundingClientRect();
    ul.style.top = `${rect.height - 1}px`;
    ul.style.width = `${rect.width - 2}px`;
  });

  function update_list_width() {
    const rect = input.getBoundingClientRect();
    ul.style.width = `${rect.width - 2}px`;
  }

  function may_complete() {
    if (to) {
      clearTimeout(to);
    }
    to = setTimeout(function () {
      complete_it();
    }, 300);
  }

  async function complete_it() {
    if (!input.value) {
      return;
    }
    selected_idx = null;
    abort.abort();
    abort = new AbortController();
    try {
      const res = await fetch(`${url}/names?g=${group}&q=${input.value}`, {
        signal: abort.signal,
      });
      const j = await res.json();
      if (!abort.signal.aborted) {
        // only update if we're current
        names = j.names;
      }
    } catch (e) {
      if (e instanceof DOMException && e.name === "AbortError") {
      } else {
        console.error(e);
      }
    }
  }

  function select_by_click(e: any) {
    let el = e.target;
    if (el.tagName === "IMG") {
      el = el.parentNode;
    }
    if (el.tagName != "LI") {
      return;
    }
    selected_idx = el.dataset.idx | 0;
    select_confirmed();
    input.focus();
    should_hide = true;
  }

  function select_confirmed() {
    selected = names[selected_idx][0];
    selected_name = names[selected_idx][1];
    input.value = selected_name;
    selected_init = null;
  }

  function update_value() {
    if (!selected || selected === selected_init) {
      return;
    }
    if (input.value) {
      input.value = selected_name;
    } else {
      selected = selected_init;
      selected_name = "";
    }
  }

  function select_by_key(e: KeyboardEvent) {
    if (e.key === "ArrowDown" || (e.key === "n" && e.altKey)) {
      select_next(1);
      e.preventDefault();
    } else if (e.key === "ArrowUp" || (e.key === "p" && e.altKey)) {
      select_next(-1);
      e.preventDefault();
    } else if (e.key === "Enter") {
      select_confirmed();
      e.preventDefault();
    }
  }

  function select_next(dir: number) {
    if (typeof selected_idx === "number") {
      if (dir > 0) {
        selected_idx = (selected_idx + 1) % names.length;
      } else {
        selected_idx = (selected_idx - 1) % names.length;
      }
    } else {
      if (dir > 0) {
        selected_idx = 0;
      } else {
        selected_idx = names.length - 1;
      }
    }
  }
</script>

<div>
  <input
    bind:this={input}
    type="text"
    on:input={() => {
      should_hide = false;
      may_complete();
    }}
    on:focus={() => (should_hide = false)}
    on:blur={() => {
      should_hide = true;
      update_value();
    }}
    on:keydown={select_by_key}
  />
  <img
    class="selected-avatar"
    alt=""
    src="{url}/avatar/{selected ? selected : 'nobody'}.jpg"
  />
  <ul
    bind:this={ul}
    on:click={select_by_click}
    on:mousedown|preventDefault={() => {}}
    class:hidden={names.length === 0 || should_hide}
  >
    {#each names as name, i (name)}
      <li data-idx={i} class:selected={i === selected_idx} title={name[1]}>
        <img src="{url}/avatar/{name[0]}.jpg" alt="avatar" />{name[1]}
      </li>
    {/each}
  </ul>
</div>

<svelte:window on:resize={update_list_width} />

<style>
  div {
    position: relative;
  }
  ul {
    list-style: none;
    padding: 0;
    margin: 0;
    position: absolute;
    left: 0;
    background-color: white;
    box-shadow: 0 0 4px var(--color-inactive);
    clip-path: polygon(-100% 0, 200% 0, 200% 200%, -100% 200%);
  }
  li {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    cursor: pointer;
    line-height: 2em;
  }
  li > img {
    height: calc(2.3em - 2px);
    width: calc(2.3em - 2px);
    vertical-align: middle;
    padding-right: 0.5em;
    /* for alt text */
    display: inline-block;
    overflow: hidden;
  }
  ul:not(:hover) > li.selected,
  li:hover {
    background-color: #d9f5ff;
  }
  input {
    padding-left: 2.5em;
    width: 100%;
  }
  input,
  ul {
    border-radius: 0;
    border: 1px solid var(--color-inactive);
  }
  input:focus,
  input:focus ~ ul {
    border-color: var(--color-active);
    box-shadow: 0 0 4px var(--color-active);
    outline: 1px solid var(--color-active);
  }
  input:focus ~ ul {
    border-top-color: var(--color-inactive);
  }
  .selected-avatar {
    position: absolute;
    pointer-events: none;
    top: 1px;
    left: 1px;
    height: calc(2.3em - 2px);
    width: calc(2.3em - 2px);
  }
  .hidden {
    display: none;
  }
</style>
