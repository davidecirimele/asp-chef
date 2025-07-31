<script context="module">
	import { Recipe } from '$lib/recipe';
	import { Utils } from '$lib/utils';
	import { Base64 } from 'js-base64';

	const operation = '@preview/SurveyJS';
	const default_extra_options = {
		predicate: '__survey__',
		input_predicate: '__input__',
		output_predicate: '__output__',
		multistage: false,
		echo: false,
		show_model_index: false,
		data: [],
		survey_data: {},
		instance_indexes: []
	};

	const listeners = new Map();

	Recipe.register_operation_type(operation, async (input, options, index, id) => {
		try {
			listeners.get(id)(input);
		} catch (error) {
			/* component not mounted, possibly because of headless mode */
		}

		const filtered = options.echo
			? input
			: input.map((part) => part.filter((atom) => atom.predicate !== options.predicate));

		if (!options.output_predicate) {
			return filtered;
		}

		return filtered.map((part, index) => {
			console.log("Processing index:", index);
			console.log("options.data[index]:", options.data[index]);

			if (options.data[index] !== null) {
				const data_length = input[index]
					.filter(atom => atom.predicate === options.predicate)
					.length;

				console.log("Data length:", data_length);

				const output_atoms = options.data[index]
					.filter((data, data_index) => {
						console.log(`Checking data at [${index}][${data_index}]`, data);
						return !!data;
					})
					.map(data =>
						Utils.parse_atom(
							`${options.output_predicate}("${Base64.encode(JSON.stringify(data))}")`
						)
					);

				console.log("Output atoms for index", index, output_atoms);

				return [...part, ...output_atoms];
			}

			return part;
		});
	});

	/*Recipe.register_operation_type(operation, async (input, options, index, id) => {
       try {
           listeners.get(id)(input);
       } catch (error) { /* component not mounted, possibly because of headless mode  }

       const result = options.echo ? input : input.map(part => part.filter(atom => atom.predicate !== options.predicate));
       console.log("RESULT", result);
       
       const res = options.data && options.data.length > 0 ? result.map(part => [...part, Utils.parse_atom(`${options.output_predicate}("${Base64.encode(JSON.stringify(options.data?.map(result => result.output).filter(result => result!=null)))}")`)]) : result;
       console.log("RES", res);
       
       return res;
   });*/
</script>

<script>
	import {
		Button,
		ButtonGroup,
		Icon,
		Input,
		InputGroup,
		InputGroupText
	} from '@sveltestrap/sveltestrap';
	import Operation from '$lib/Operation.svelte';
	import { onDestroy, onMount } from 'svelte';
	import SurveyJS from './+SurveyJS.svelte';
	import { merge } from 'vega';
	import Popover from '$lib/Popover.svelte';

	export let id;
	export let options;
	export let index;
	export let add_to_recipe;
	export let keybinding;

	let models = [];
	let jsons = {};

	function populate_input(input) {
		jsons = Utils.extract_json_objects(input, options.input_predicate);

		try{
			Object.values(jsons).forEach((jsonArray, idx) => {
				const model_index = idx + 1;
				if(!options.survey_data[model_index])
					options.survey_data[model_index] = {}
				jsonArray.forEach((json, i) => {
					const exists = Object.values(options.survey_data[model_index] || {}).some((existing) =>
						Utils.compare_jsons(existing.input, json)
					);

					if (!exists) {
						options.survey_data[model_index][i+1] = {
							predicate: options.input_predicate,
							input: json,
							value: json,
						};
					}
				});
			});
		}catch(err){
			console.log(err);
		}
		console.log("SURVEYS", options.survey_data);
	}

	function init_data(models) {
		console.log("MODELLI", models);
		models.forEach((model, idx) => {
			console.log("INDEX", idx);
			if(!options.data[idx]){
				options.data[idx] = [];
			}
			if(!options.instance_indexes[idx]){
				options.instance_indexes[idx] = 1;
			}
			const model_index = idx + 1;
			try{
				if (!options.survey_data[model_index][1]) {
					options.survey_data[model_index][1] = {
						predicate: options.input_predicate,
						input: null,
						value: null,
					};
				}
			}catch(err){
				console.log(err);
			}		
		});
		console.log("OPTIONS.DATA INIT", options.data);
	}
		
		/*
		for (let i = 0; i < options.survey_data.length; i++) {
			let model = options.survey_data[i];

			try{
				options.survey_data[i] = model
					.filter(
						(json) =>
							Object.values(jsons).flat().some((new_json) => Utils.compare_jsons(json.input, new_json)) ||
							(json.input === null && json.predicate === options.input_predicate)
					)
					.sort((a, b) => {
						if (a.input === null && b.input !== null) return 1;
						if (a.input !== null && b.input === null) return -1;
						return 0;
					});

				if (options.survey_data[i].length === 0) {
					options.survey_data[i].push({
						predicate: options.input_predicate,
						input: null,
						value: null,
					});
					if(!options.instance_indexes[i])
						options.instance_indexes[i] = [];
					options.instance_indexes[i].push(1);
				}

				options.survey_data[i] = model;
			}catch(err){
				console.log(err);
			}
		}*/

		
	

	function edit() {
		Recipe.edit_operation(id, index, options);
	}

	function set_data(model_index, configuration_index, data) {
		console.log("DATI DA SALVARE", data);
		
		Object.values(options.survey_data[model_index + 1]).forEach((item, index) => {
			options.data[model_index][index] = item.value;
		});

		options.data[model_index][configuration_index-1] = data;
		options.survey_data[model_index + 1][configuration_index].value = data;

		console.log("SURVEYS SET DATA", options.survey_data);
		console.log("DATA SET DATA", options.data);
	}

	onMount(() => {
		listeners.set(id, (input) => {
			models = input;
			populate_input(models);
			init_data(models);
		});
	});

	onDestroy(() => {
		listeners.delete(id);
	});
</script>

<Operation {id} {operation} {options} {index} {default_extra_options} {add_to_recipe} {keybinding}>
	<InputGroup>
		<InputGroupText>Predicate</InputGroupText>
		<Button
			outline={!options.multistage}
			on:click={() => {
				options.multistage = !options.multistage;
				edit();
			}}>Multi-Stage</Button
		>
		<Input
			type="text"
			placeholder="predicate"
			bind:value={options.predicate}
			on:input={edit}
			data-testid="SurveyJS-predicate"
		/>
		<Button
			outline={!options.echo}
			on:click={() => {
				options.echo = !options.echo;
				edit();
			}}>Echo</Button
		>
		<Button
			outline={!options.show_model_index}
			on:click={() => {
				options.show_model_index = !options.show_model_index;
				edit();
			}}>Model Index</Button
		>
	</InputGroup>
	<InputGroup>
		<InputGroupText>Input Predicate</InputGroupText>
		<Input
			type="text"
			placeholder="predicate"
			bind:value={options.input_predicate}
			on:input={edit}
			data-testid="SurveyJS-Inputpredicate"
		/>
		<InputGroupText>Output Predicate</InputGroupText>
		<Input
			type="text"
			placeholder="predicate"
			bind:value={options.output_predicate}
			on:input={edit}
			data-testid="SurveyJS-Outputpredicate"
		/>
	</InputGroup>
	<div slot="output">
		<div class="m-1" style="overflow-y: auto;">
			{#each models as model, model_index}
				{#if options.show_model_index}
					<h6 class="text-center">Model #{model_index + 1}</h6>
				{/if}
				{#key model}
					{#each model.filter((atom) => atom.predicate === options.predicate) as configuration, configuration_index}

					<InputGroup>
						<InputGroupText>Instance #</InputGroupText>
						<Input 
							type="number"
							bind:value={options.instance_indexes[model_index]}
							min="1"
							max={Math.max(...Object.keys(options.survey_data[model_index + 1] || {}).map(Number))}
							style="max-width: 5em;"
						/>
						<InputGroupText style={'flex: 1'}>of {options.survey_data[model_index + 1] ? Math.max(...Object.keys(options.survey_data[model_index + 1] || {}).map(Number)) : 0}</InputGroupText>
						<ButtonGroup>
							<Popover title="Jump to instance #1" value="Jump to the first instance of the Survey">
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										options.instance_indexes[model_index] = 1;
										edit();
									}}><Icon name="chevron-double-left" /></Button
								>
							</Popover>
							<Popover title="Go to left" value="Move to the previous instance of the Survey">
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										options.instance_indexes[model_index] -= 1;
										edit();
									}}><Icon name="arrow-left" /></Button
								>
							</Popover>
							<Popover title="Go to right" value="Move to the next instance of the Survey">
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										options.instance_indexes[model_index] += 1;
										edit();
									}}><Icon name="arrow-right" /></Button
								>
							</Popover>
							<Popover title="Jump to the last instance" value="Jump to the last instance of the Survey">
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										options.instance_indexes[model_index] = Math.max(...Object.keys(options.survey_data[model_index + 1] || {}).map(Number));
										edit();
									}}><Icon name="chevron-double-right" /></Button
								>
							</Popover>
						</ButtonGroup>
						<ButtonGroup>
							<Popover title="Add instance" value="Add a new instance to the Survey">
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										options.survey_data[model_index+1][Math.max(...Object.keys(options.survey_data[model_index + 1] || {}).map(Number))+1] = {
											predicate: options.input_predicate,
											input: null,
											value: null,
										};
										edit();
									}}><Icon name="plus-lg" /></Button
								>
							</Popover>
							<Popover
								title="Remove instance"
								value="Remove instance #{configuration_index+1} from the Survey"
							>
								<Button
									size="lg"
									outline={true}
									on:click={() => {
										
										edit();
									}}><Icon name="trash" /></Button
								>
							</Popover>
						</ButtonGroup>
					</InputGroup>
						
						<SurveyJS
							input={options.survey_data[model_index + 1]?.[options.instance_indexes[model_index]]?.value}
							part={model}
							{index}
							configuration_atom={configuration}
							multistage={options.multistage}
							on_ok={(data) => {
								set_data(model_index , options.instance_indexes[model_index], data);
								edit();
							}}
							on_clear={() => {
								set_data(model_index , options.instance_indexes[model_index], null);
								edit();
							}}
						/>
					{/each}
				{/key}
			{/each}
		</div>
	</div>
</Operation>
