<script>
	import { createEventDispatcher } from 'svelte';
	import { openModal } from 'yasp-modals';

	import { create, convertPercentageToUint } from '../../../../sdk/';

	import { currentNetwork } from '$lib/modules/network';
	import { connected, getSigner } from '$lib/modules/wallet';

	import AddRecipientModal from './Modals/AddRecipientModal.svelte';
	import OnlyKnownNetwork from './Network/OnlyKnownNetwork.svelte';
	import { isAddressValid, shortenAddress } from '$lib/utils/utils';
	import OnlyConnected from './OnlyConnected.svelte';
	import Loading from './Loading.svelte';

	const dispatch = createEventDispatcher();

	let name = '';
	let recipients = [];
	let loading = false;

	let mustReset = false;

	$: sum = recipients.reduce((acc, recipient) => acc + recipient.percent, 0);
	$: invalidRecipients = recipients.filter((r) => !isAddressValid(r.account));
	$: recipientsWithAllocation = recipients.filter((r) => r.percent !== 0);

	$: canSubmit =
		$connected &&
		name.trim().length > 0 &&
		sum == 100 &&
		invalidRecipients.length == 0 &&
		recipientsWithAllocation.length > 1;

	function initRecipients() {
		recipients = [];
	}

	async function onSubmit() {
		const withAllocation = recipients
			.filter((r) => r.percent !== 0)
			.map((r) => ({ account: r.account, percent: convertPercentageToUint(r.percent) }));
		const diff = withAllocation.length - recipients.length;

		if (
			diff == 0 ||
			confirm('Some recipients do not have any allocation and will be removed. Are you sure?')
		) {
			loading = true;
			const events = await create(
				name,
				withAllocation,
				await getSigner(),
				$currentNetwork.factory_address
			)
				.then((tx) => tx.wait())
				.then((receipt) => {
					return receipt;
				})
				.then((receipt) => receipt.events)
				.catch((err) => {
					console.error(err);
					alert(`Failed to create Splitter contract: \n ${err.message}`);
				});

			loading = false;
			mustReset = true;
			dispatch('splitter', {
				address: events[0].args[0],
				name
			});
		}
	}

	function onReset() {
		initRecipients();
		mustReset = false;
	}

	function addLine() {
		openModal(AddRecipientModal, {
			callback: (recipient) => {
				recipients.push(recipient);
				recipients = recipients;
			}
		});
	}

	function editLine(index) {
		openModal(AddRecipientModal, {
			account: recipients[index].account,
			percent: recipients[index].percent,
			callback: (recipient) => {
				recipients[index] = recipient;
				recipients = recipients;
			}
		});
	}

	function removeLine(index) {
		recipients.splice(index, 1);
		recipients = [...recipients];
	}

	initRecipients();
</script>

<form on:submit|preventDefault>
	<div class="form__group form__name">
		<label for="name"><strong>Splitter name *</strong></label>
		<input
			type="text"
			bind:value={name}
			id="name"
			placeholder="Choose a name to identify the collab splitter"
		/>
	</div>
	<div class="form__group form__recipients">
		<strong>Recipients *</strong>
		{#if recipients.length}
			<table>
				<thead>
					<tr>
						<th scope="col" />
						<th scope="col">Address</th>
						<th scope="col">Allocation</th>
						<th scope="col" />
					</tr>
				</thead>
				<tbody>
					{#each recipients as recipient, index}
						<tr class="form__recipients__row">
							<td><label for="recipient-{index}">#{index + 1}</label></td>
							<td class="form__recipients__row__address">
								{shortenAddress(recipient.account)}
							</td>
							<td class="form__recipients__row__allocation">
								{recipient.percent}%
							</td>
							<td>
								<button on:click={() => editLine(index)} class="form__recipient__remove"
									>edit</button
								>
								<button on:click={() => removeLine(index)} class="form__recipient__remove light"
									>remove</button
								>
							</td>
						</tr>
					{/each}
				</tbody>
			</table>
		{:else}
			<p class="empty">No recipient yet!</p>
		{/if}
	</div>
	<div class="form__group form__add">
		<button type="button" on:click={addLine}>Add recipient</button>
	</div>
	<p class="form__group form__total">
		Total: <b>{sum}%</b>
		<span class="form__error">
			{#if sum > 100}
				Error: total is greater than 100%.
			{:else if sum < 100}
				<em>
					({100 - sum}% left to split)
				</em>
			{/if}
		</span>
	</p>
	<p>
		<b>{recipients.length}</b> recipients.
	</p>

	<OnlyConnected>
		<OnlyKnownNetwork>
			{#if !mustReset}
				<button on:click={onSubmit} disabled={!canSubmit} class="form__create">
					{#if loading}
						<Loading>
							<p>Creating...</p>
						</Loading>
					{:else}
						Create
					{/if}
				</button>
			{:else}
				<button on:click={onReset} class="form__create">Reset</button>
			{/if}
		</OnlyKnownNetwork>
	</OnlyConnected>
</form>

<style lang="postcss">
	form {
		@apply max-w-screen-md w-full mx-auto;
	}

	.form__name {
		@apply flex flex-col mt-4;
	}

	table {
		border-collapse: separate;
		border-spacing: 0 0.5em;
		width: 100%;
		text-align: center;
	}

	th,
	td {
		@apply px-2;
	}

	.form__group {
		@apply py-2;
	}

	.form__name input {
		@apply mt-2;
	}

	.form__recipients {
		@apply mt-4;
	}

	.form__recipients__row {
	}

	.form__recipient__remove {
		@apply ml-6 rounded px-2 py-1;
	}

	.form__add {
		@apply text-center;
	}

	.form__error {
		@apply text-red-400;
	}

	.form__total {
		@apply mt-4;
	}

	.form__create {
		@apply w-full mt-4 flex justify-center;
	}

	input {
		@apply px-1 py-1;
		color: var(--secondary);
	}

	.empty {
		@apply py-6 text-center font-medium;
	}
</style>
