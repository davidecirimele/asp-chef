<script context="module">
    import {Recipe} from "$lib/recipe";
    import _ from 'lodash';

    const operation = "Sort by Predicate or Argument";
    const default_extra_options = {
        sort_index: 0,
        descending: false,
    };

    Recipe.register_operation_type(operation, async (input, options) => {
        const comparator = options.sort_index === 0 ?
            [atom => atom.predicate] :
            [atom => {
                if (!atom.terms) {
                    return undefined;
                }
                const term = atom.terms[options.sort_index - 1];
                if (term === undefined) {
                    return undefined
                } else if (term.number !== undefined) {
                    return term.number;
                } else {
                    return term.str;
                }
            }];
        const mapper = options.descending ?
            model => _.sortBy(model, comparator).reverse() :
            model => _.sortBy(model, comparator);
        return input.map(mapper);
    });
</script>

<script>
    import Operation from "$lib/Operation.svelte";
    import {Button, Input, InputGroup, InputGroupText} from "@sveltestrap/sveltestrap";

    export let id;
    export let options;
    export let index;
    export let add_to_recipe;
    export let keybinding;

    function edit() {
        Recipe.edit_operation(id, index, options);
    }

    function toggle_descending() {
        options.descending = !options.descending;
        edit();
    }

</script>

<Operation {id} {operation} {options} {index} {default_extra_options} {add_to_recipe} {keybinding}>
    <InputGroup>
        <InputGroupText>Index</InputGroupText>
        <Input type="number"
               min="0"
               bind:value={options.sort_index}
               on:input={edit}
               data-testid="SortByPredicateOrArgument-sort-index"
        />
        <Button outline="{!options.descending}" on:click={toggle_descending}>Descending</Button>
    </InputGroup>
</Operation>
