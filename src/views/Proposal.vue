<template>
    <Container :slim="true">
        <div class="px-4 px-md-0 mb-3">
            <router-link :to="{name: domain ? 'home' : 'proposals'}" class="text-gray">
                <Icon name="back" size="22" class="v-align-middle" />
                {{ space.name }}
            </router-link>
        </div>
        <div>
            <div class="col-12 col-lg-8 float-left pr-0 pr-lg-5">
                <div class="px-4 px-md-0">
                    <template v-if="loaded">
                        <h1 class="mb-2">
                            {{ title }}
                            <span v-text="`#${id.slice(0, 7)}`" class="text-gray" />
                        </h1>
                        <State :proposal="proposal" class="mb-4" />
                        <!-- <UiMarkdown :body="payload.body" class="mb-6" /> -->
                    </template>
                    <PageLoading v-else />
                </div>
                <Block v-if="loaded && ts >= payload.start && ts < payload.end && !receipt.hasVoted" class="mb-4" title="Cast your vote">
                    <div class="mb-3">
                        <UiButton
                            v-for="(choice, i) in payload.choices"
                            :key="i"
                            @click="selectedChoice = i + 1"
                            class="d-block width-full mb-2"
                            :class="selectedChoice === i + 1 && 'button--active'"
                        >
                            {{ choice }}
                            <a
                                v-if="_get(payload, `metadata.plugins.aragon.choice${i + 1}`)"
                                @click="modalOpen = true"
                                :aria-label="
                                    `Target address: ${payload.metadata.plugins.aragon[`choice${i + 1}`].actions[0].targetAddress}\nCalldata: ${
                                        payload.metadata.plugins.aragon[`choice${i + 1}`].actions[0].calldata
                                    }`
                                "
                                class="tooltipped tooltipped-n break-word"
                            >
                                <Icon name="warning" class="v-align-middle ml-1" />
                            </a>
                        </UiButton>
                    </div>
                    <UiButton
                        :disabled="voteLoading || !selectedChoice || !web3.account"
                        :loading="voteLoading"
                        @click="modalOpen = true"
                        class="d-block width-full button--submit"
                    >
                        Vote
                    </UiButton>
                </Block>
                <BlockVotes v-if="loaded" :space="space" :proposal="proposal" :votes="votes" />
            </div>
            <div v-if="loaded" class="col-12 col-lg-4 float-left">
                <Block title="Information">
                    <!-- <div class="mb-1 overflow-hidden">
            <b>Token(s)</b>
            <a
              @click="modalStrategiesOpen = true"
              class="float-right text-white"
            >
              <span v-for="(symbol, symbolIndex) of symbols" :key="symbol">
                <Token :space="space.key" :symbolIndex="symbolIndex" />
                {{ symbol }}
                <span
                  v-show="symbolIndex !== symbols.length - 1"
                  v-text="'+'"
                  class="mr-1"
                />
              </span>
            </a>
          </div> -->
                    <div class="mb-1">
                        <b>Author</b>
                        <User :address="proposal.address" :space="space" class="float-right" />
                    </div>
                    <div class="mb-1">
                        <b>ID</b>
                        <a class="float-right">
                            #{{ proposal.id }}
                            <!-- <Icon name="external-link" class="ml-1" /> -->
                        </a>
                    </div>
                    <div>
                        <div class="mb-1">
                            <b>Start date</b>
                            <span
                                :aria-label="_ms(payload.start)"
                                v-text="$d(payload.start * 1e3, 'short')"
                                class="float-right text-white tooltipped tooltipped-n"
                            />
                        </div>
                        <div class="mb-1">
                            <b>End date</b>
                            <span
                                :aria-label="_ms(payload.end)"
                                v-text="$d(payload.end * 1e3, 'short')"
                                class="float-right text-white tooltipped tooltipped-n"
                            />
                        </div>
                        <!-- <div class="mb-1">
              <b>Snapshot</b>
              <a
                :href="_explorer(space.network, payload.snapshot, 'block')"
                target="_blank"
                class="float-right"
              >
                {{ $n(payload.snapshot) }}
                <Icon name="external-link" class="ml-1" />
              </a>
            </div> -->
                    </div>
                </Block>
                <BlockResults :id="id" :space="space" :payload="payload" :results="results" :votes="votes" />
            </div>
        </div>
        <ModalConfirm
            v-if="loaded"
            :open="modalOpen"
            @close="modalOpen = false"
            @reload="loadProposal"
            :space="space"
            :proposal="proposal"
            :id="id"
            :selectedChoice="selectedChoice"
            :totalScore="totalScore"
            :scores="scores"
            :snapshot="payload.snapshot"
            :delegatee="delegatee"
        />
        <ModalStrategies :open="modalStrategiesOpen" @close="modalStrategiesOpen = false" :space="space" :strategies="space.strategies" />
    </Container>
</template>

<script>
import {mapActions} from 'vuex';

export default {
    data() {
        return {
            key: 'symblox',
            id: this.$route.params.id,
            loading: false,
            loaded: false,
            voteLoading: false,
            proposal: {},
            votes: {},
            results: [],
            receipt: {},
            hasVoted: false,
            modalOpen: false,
            modalStrategiesOpen: false,
            selectedChoice: 0,
            totalScore: 0,
            delegatee: '',
            scores: []
        };
    },
    computed: {
        space() {
            return this.app.spaces[this.key];
        },
        payload() {
            return this.proposal.msg ? this.proposal.msg.payload : {};
        },
        title () {
            return this.$route.query.name || '';
        },
        ts() {
            return (Date.now() / 1e3).toFixed();
        },
        symbols() {
            return this.space.strategies.map(strategy => strategy.params.symbol);
        }
    },
    watch: {
        'web3.account': async function(val, prev) {
            if (val && val.toLowerCase() !== prev){
                this.loading = true;
                this.loaded = false;
                await this.loadPower();
                await this.loadProposal();
                this.loading = false;
                this.loaded = true;
            }
        }
    },
    methods: {
        ...mapActions(['getProposal', 'getPower', 'send', 'getReceipt','getDelegatee']),
        async loadProposal() {
            this.name = "fff";
            const proposalObj = await this.getProposal({
                space: this.space,
                id: this.id,
                name: this.name
            });
            this.receipt = await this.getReceipt({
                space: this.space,
                id: this.id,
                name: this.name
            });
            this.proposal = proposalObj.proposal;
            this.votes = proposalObj.votes;
            this.results = proposalObj.results;
            this.hasVoted = proposalObj.hasVoted;
        },
        async loadPower() {
            if (!this.web3.account) return;
            const {scores, totalScore} = await this.getPower({
                space: this.space,
                address: this.web3.account
                // snapshot: this.payload.snapshot
            });
            this.totalScore = totalScore || 0;
            this.scores = scores || [];
            this.delegatee = await this.getDelegatee(this.space);
        }
    },
    async created() {
        this.loading = true;
        await this.loadProposal();
        await this.loadPower();
        this.loading = false;
        this.loaded = true;
    }
};
</script>
