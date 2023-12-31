<script lang="ts">
    import AuthCheck from '$lib/components/AuthCheck.svelte';
	import type { Interaction } from '$lib/model.js';
    import { API_URL, refreshSingleEvaluation } from '$lib/utility';
	import { onDestroy, onMount } from 'svelte';
    import 'c3/c3.css';
    import io from 'socket.io-client';
    import Card from '$lib/components/landing/components/Card.svelte';
	import Cards from '$lib/components/landing/Cards.svelte';
    
    export let data;
    let { campaign } = data;
    let interactions: Interaction[];
    let responded_interactions: Interaction[] = [];
    let funnelLoading = false;
    let insightsLoading = false;
    let summaries = [
        { title: 'Campaign Manager Summary', content: campaign.campaign_manager_summary },
        { title: 'Communications Summary', content: campaign.communications_director_summary },
        { title: 'Field Summary', content: campaign.field_director_summary }
    ];
    let socket;

    async function refreshEvaluations() {
        //for each item in the responded interactions array call the refreshSingleEvaluation function
        responded_interactions.forEach(interaction => {
            refreshSingleEvaluation(campaign.id, interaction.id);
            socket.emit('subscribe_interaction_evaluation', { "interaction_id": interaction.id });
            console.log(`Emitted subscribe_interaction_evaluation for interaction id: , ${interaction.id}`)
            interaction.loading = true;
            //trigger reactivity
            responded_interactions = [...responded_interactions];
        });
    }

    // Modify the function to accept an event object
    async function singleRefreshRequest(event: MouseEvent) {
        const interaction_id = event?.currentTarget?.dataset?.interactionId;

        if(!interaction_id) {
            console.log('No interaction ID found');
            return;
        }

        refreshSingleEvaluation(campaign.id, interaction_id)
        socket.emit('subscribe_interaction_evaluation', { "interaction_id": interaction_id });
        console.log(`Emitted subscribe_interaction_evaluation for interaction id: , ${interaction_id}`)
        const interaction = responded_interactions.find(interaction => interaction.id == interaction_id);
        if(!interaction) {
            console.log(responded_interactions)
            console.log('Interaction not found');
            return;
        }
        interaction.loading = true;

        //trigger reactivity
        responded_interactions = [...responded_interactions];
    }


    async function refreshFunnel() {

        funnelLoading = true;
        socket.emit('subscribe_funnel_refresh', { "campaign_id": campaign.id });
        console.log(`Emitted subscribe_funnel_refresh for campaign id: , ${campaign.id}`)
        const res = await fetch(`${API_URL}/campaign/insights`, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ 
                campaign_id: campaign.id,
                refresh_funnel: true })
        });
        const data = await res.json();
        console.log(data);
    }

    async function refreshInsights() {

        insightsLoading = true;
        socket.emit('subscribe_campaign_insight_refresh', { "campaign_id": campaign.id });
        const res = await fetch(`${API_URL}/campaign/insights`, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ 
                campaign_id: campaign.id,
                refresh_campaign_insights: true })
        });
        const data = await res.json();
        console.log(data);
    }

    async function renderChart() {

        const object = await import('c3');
        const c3 = object.default;
        const categories = ['Sent', 'Delivered', 'Responded', 'Converted'];

        var chart = c3.generate({
            bindto: '#funnelChart', // make sure to bind to an element with this ID
            data: {
                columns: [
                    ['Campaign', campaign.interactions_sent, campaign.interactions_delivered, campaign.interactions_responded, campaign.interactions_converted]
                ],
                colors: {
                    Campaign: '#FFFFFF'
                },
                type: 'bar'
            },
            bar: {
                width: {
                    ratio: 0.5
                }
            },
            axis: {
                x: {
                    type: 'category',
                    categories: ['Sent', 'Delivered', 'Responded', 'Converted']
                }
            },
            tooltip: {
                format: {
                    title: function (d) { return categories[d]; },
                    name: function (name, ratio, id, index) { return ""; },
                    value: function (value, ratio, id, index) {
                        return value;
                    }
                }
            }
        });
        console.log(chart);
    }

    onMount(async () => {
        console.log(campaign);

        socket = io(`${API_URL}`);

        //log all inbound socket events
        socket.onAny((event, ...args) => {
            console.log(event, args);
        });

        socket.on("interaction_evaluated", async (data) => {
            console.log("Received interaction_evaluated event");
            console.log(data);

            // Find the interaction in the array
            const interaction = responded_interactions.find(interaction => interaction.id === data.interaction_id);
            if (!interaction) {
                console.log("Interaction not found");
                return;
            }

            // Fetch the interaction object from the API
            const response = await fetch(`${API_URL}/interaction?interaction_id=${data.interaction_id}`);
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            const interaction_response = await response.json();

            // Update the interaction object
            Object.assign(interaction, interaction_response.interaction);
            interaction.loading = false;

            // This assignment will trigger Svelte's reactivity
            responded_interactions = [...responded_interactions];
        });

        //if I get an error from the socket route, I should set the interaction id to not loading
        socket.on("interaction_evaluation_error", async (data) => {
            console.log("Received error event");
            console.log(data);

            // Find the interaction in the array
            const interaction = responded_interactions.find(interaction => interaction.id === data.interaction_id);
            if (!interaction) {
                console.log("Interaction not found");
                return;
            }

            interaction.loading = false;

            // This assignment will trigger Svelte's reactivity
            responded_interactions = [...responded_interactions];
        });
        
        socket.on("funnel_refreshed", async (data) => {
            console.log("Received funnel_refreshed event");
            console.log(data);

            // Fetch the campaign object from the API
            const response = await fetch(`${API_URL}/campaign?campaign_id=${campaign.id}`);
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            const campaign_response = await response.json();

            // Update the campaign object
            Object.assign(campaign, campaign_response.campaign);

            // This assignment will trigger Svelte's reactivity
            funnelLoading = false;
            renderChart();
            campaign = campaign;
        });

        socket.on("campaign_insight_refreshed", async (data) => {
            console.log("Received campaign_insight_refreshed event");
            console.log(data);

            // Fetch the campaign object from the API
            const response = await fetch(`${API_URL}/campaign?campaign_id=${campaign.id}`);
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            const campaign_response = await response.json();

            // Update the campaign object
            Object.assign(campaign, campaign_response.campaign);

            // This assignment will trigger Svelte's reactivity
            campaign = campaign;
            insightsLoading = false;
        });

        // Get interactions from ${API_URL}/interaction?campaign_id=${campaign.id}
        const res = await fetch(`${API_URL}/interaction?campaign_id=${campaign.id}`);
        const data = await res.json();
        console.log(data);
        interactions = data.interactions;

        console.log(summaries)

        // Filter for interactions where campaign_relevance_score is not null
        responded_interactions = interactions.filter(interaction => interaction.interaction_status >= 6); // conversation_status >= 6 means the voter has responded to the campaign
        responded_interactions.sort((a, b) => b.campaign_relevance_score - a.campaign_relevance_score);
        console.log(responded_interactions);

        renderChart();

        return () => {
            socket?.disconnect();
        };
    });
</script>

<div class="px-20">
    <AuthCheck>
        
        <!-- Three buttons for "Refresh Evaluations", "Refresh Funnel", "Refresh Insights"-->
        
        <div class="w-full">
        <h1>Goal</h1>
        {#if funnelLoading}
            <span class="loading loading-spinner loading-lg h-max"></span>
        {:else}
            <div class="">
                <div class="text-5xl">{campaign.campaign_goal}</div>
            </div>
            <div class="flex justify-between h-max pt-5">
                <div class="w-1/3 flex flex-col justify-center items-center">
                    <div class="flex flex-col justify-between h-full">
                        <div class="h-1/2">
                            <h2 class="text-8xl">{campaign.interactions_sent}</h2>
                            <p>Messages Sent</p>
                        </div>
                        <div class=" h-1/2">
                            <h2 class="text-8xl">{campaign.interactions_converted}</h2>
                            <p>Converted</p>
                        </div>
                    </div>
                </div>
                <div id="funnelChart" class="w-2/3 h-full"></div>
            </div>
        {/if}
        <btn class="btn btn-primary" on:click={refreshFunnel}>Refresh Funnel</btn>
        </div>

        <div class="pt-10">
            {#if insightsLoading}
                <span class="loading loading-spinner loading-lg h-max"></span>
            {:else} 
                <!-- Check if campaign.policy_insights is a string. If so, do not show it -->
                {#if campaign.policy_insights && typeof campaign.policy_insights === 'object' && !(campaign.policy_insights instanceof String)}
                    
                <div class="pt-5"><Cards title="Policy Insights" cards={Object.entries(campaign.policy_insights).map(([policy_area, insight]) => ({ title: policy_area, content: insight }))}/></div>
                {/if}
            {/if}
            <button class="btn btn-primary mt-4" on:click={refreshInsights}>Refresh Insights</button>
        </div>
        

        
        <div class="pt-10">
            <table class="min-w-full bg-base-100 text-base table table-zebra">
                <thead>
                    <tr>
                        <th class="px-4 py-2">Voter</th>
                        <th class="px-4 py-2">Summary</th>
                        <th class="px-4 py-2">Policy Insights</th>
                        <th class="px-4 py-2">Voter Insights</th>
                        <th class="px-4 py-2">Conversation</th>
                    </tr>
                </thead>
                <tbody>
                    {#each responded_interactions as interaction}
                        <tr>
                            {#if interaction.loading}
                                <span class="loading loading-spinner loading-md h-28"></span>
                            {:else}
                                <td class="px-4 py-2"><div class="h-28 overflow-y-auto">{interaction.voter.voter_name} ({interaction.voter.voter_phone_number})</div></td>
                                <td class="px-4 py-2"><div class="h-28 overflow-y-auto">{interaction.campaign_relevance_summary}</div></td>
                                <td class="px-4 py-2">
                                    <div class="h-28 overflow-y-auto">
                                        {#if interaction.insights_about_issues && typeof interaction.insights_about_issues === 'object' && !(interaction.insights_about_issues instanceof String)}
                                            {#each Object.entries(interaction.insights_about_issues) as [policy_area, insight]}
                                                <p><span class="font-semibold">{policy_area}</span>: {insight}</p>
                                            {/each}
                                        {:else}
                                            No Policy Insights
                                        {/if}
                                    </div>
                                </td>
                                <td class="px-4 py-2"><div class="h-28 overflow-y-auto">{interaction.insights_about_voter}</div></td>
                                <td class="px-4 py-2 btn btn-primary m-2"><a href="/interaction/{interaction.id}">View Conversation</a></td> 
                                <!-- Add a data attribute to store the interaction ID -->
                                <td class=" btn btn-primary mt-2 mx-2" data-interaction-id={interaction.id} on:click={singleRefreshRequest}>Refresh Evaluation</td>
                            {/if}
                        </tr>
                    {/each}
                </tbody>
            </table>
            <button class="btn btn-primary my-5" on:click={refreshEvaluations}>Refresh Evaluations</button>
        </div>


    </AuthCheck>
</div>

<style>
    :global(.c3-tooltip-container .c3-tooltip-name--Campaign-Funnel),
    :global(.c3-tooltip-container .name),
    :global(.c3-tooltip-container .value) {
        color: #000000 !important;
    }
</style>