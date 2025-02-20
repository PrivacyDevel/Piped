<template>
    <ErrorHandler v-if="channel && channel.error" :message="channel.message" :error="channel.error" />

    <div v-if="channel" v-show="!channel.error">
        <img v-if="channel.bannerUrl" :src="channel.bannerUrl" class="w-full pb-1.5" loading="lazy" />
        <div class="pp-channel-page-author flex place-items-center">
            <img height="48" width="48" class="m-1" :src="channel.avatarUrl" />
            <h5 v-text="channel.name" />
            <font-awesome-icon class="ml-1.5" v-if="channel.verified" icon="check" />
        </div>
        <!-- eslint-disable-next-line vue/no-v-html -->
        <p class="whitespace-pre-wrap mt-2">
            <span v-html="purifyHTML(urlify(channel.description))" />
        </p>

        <div class="flex mt-4 mb-2 pp-channel-tabs">
            <button
                class="btn pp-subscribe"
                @click="subscribeHandler"
                v-t="{
                    path: `actions.${subscribed ? 'unsubscribe' : 'subscribe'}`,
                    args: { count: numberFormat(channel.subscriberCount) },
                }"
            ></button>

            <!-- RSS Feed button -->
            <a
                aria-label="RSS feed"
                title="RSS feed"
                role="button"
                v-if="channel.id"
                :href="`${apiUrl()}/feed/unauthenticated/rss?channels=${channel.id}`"
                target="_blank"
                class="btn"
                style="display: inline; float: unset; margin-left: var(--efy_gap0)"
            >
                <font-awesome-icon icon="rss" />
            </a>
            <WatchOnYouTubeButton :link="`https://youtube.com/channel/${this.channel.id}`" />
            <p>|</p>
            <button
                v-for="(tab, index) in tabs"
                :key="tab.name"
                style="margin-right: var(--efy_gap0)"
                @click="loadTab(index)"
                :class="{ active: selectedTab == index }"
            >
                <span v-text="tab.translatedName"></span>
            </button>
        </div>

        <hr />

        <div class="video-grid">
            <ContentItem
                v-for="item in contentItems"
                :key="item.url"
                :item="item"
                height="94"
                width="168"
                hide-channel
                class="efy_trans_filter"
            />
        </div>
    </div>
</template>

<style>
.pp-channel-tabs > p {
    place-self: center;
    padding: 0 10rem;
    -webkit-text-fill-color: var(--efy_text_trans2);
}
</style>

<script>
import ErrorHandler from "./ErrorHandler.vue";
import ContentItem from "./ContentItem.vue";
import WatchOnYouTubeButton from "./WatchOnYouTubeButton.vue";

export default {
    components: {
        ErrorHandler,
        ContentItem,
        WatchOnYouTubeButton,
    },
    data() {
        return {
            channel: null,
            subscribed: false,
            tabs: [],
            selectedTab: 0,
            contentItems: [],
        };
    },
    mounted() {
        this.getChannelData();
    },
    activated() {
        if (this.channel && !this.channel.error) document.title = this.channel.name + " - Piped";
        window.addEventListener("scroll", this.handleScroll);
        if (this.channel && !this.channel.error) this.updateWatched(this.channel.relatedStreams);
    },
    deactivated() {
        window.removeEventListener("scroll", this.handleScroll);
    },
    unmounted() {
        window.removeEventListener("scroll", this.handleScroll);
    },
    methods: {
        async fetchSubscribedStatus() {
            if (!this.channel.id) return;
            if (!this.authenticated) {
                this.subscribed = this.isSubscribedLocally(this.channel.id);
                return;
            }

            this.fetchJson(
                this.authApiUrl() + "/subscribed",
                {
                    channelId: this.channel.id,
                },
                {
                    headers: {
                        Authorization: this.getAuthToken(),
                    },
                },
            ).then(json => {
                this.subscribed = json.subscribed;
            });
        },
        async fetchChannel() {
            const url = this.apiUrl() + "/" + this.$route.params.path + "/" + this.$route.params.channelId;
            return await this.fetchJson(url);
        },
        async getChannelData() {
            this.fetchChannel()
                .then(data => (this.channel = data))
                .then(() => {
                    if (!this.channel.error) {
                        document.title = this.channel.name + " - Piped";
                        this.contentItems = this.channel.relatedStreams;
                        this.fetchSubscribedStatus();
                        this.updateWatched(this.channel.relatedStreams);
                        this.tabs.push({
                            translatedName: this.$t("video.videos"),
                        });
                        for (let i = 0; i < this.channel.tabs.length; i++) {
                            let tab = this.channel.tabs[i];
                            tab.translatedName = this.getTranslatedTabName(tab.name);
                            this.tabs.push(tab);
                        }
                    }
                });
        },
        handleScroll() {
            if (
                this.loading ||
                !this.channel ||
                !this.channel.nextpage ||
                (this.selectedTab != 0 && !this.tabs[this.selectedTab].tabNextPage)
            )
                return;
            if (window.innerHeight + window.scrollY >= document.body.offsetHeight - window.innerHeight) {
                this.loading = true;
                if (this.selectedTab == 0) {
                    this.fetchChannelNextPage();
                } else {
                    this.fetchChannelTabNextPage();
                }
            }
        },
        fetchChannelNextPage() {
            this.fetchJson(this.apiUrl() + "/nextpage/channel/" + this.channel.id, {
                nextpage: this.channel.nextpage,
            }).then(json => {
                this.channel.nextpage = json.nextpage;
                this.loading = false;
                this.updateWatched(json.relatedStreams);
                json.relatedStreams.map(stream => this.contentItems.push(stream));
            });
        },
        fetchChannelTabNextPage() {
            this.fetchJson(this.apiUrl() + "/channels/tabs", {
                data: this.tabs[this.selectedTab].data,
                nextpage: this.tabs[this.selectedTab].tabNextPage,
            }).then(json => {
                this.tabs[this.selectedTab].tabNextPage = json.nextpage;
                this.loading = false;
                json.content.map(item => this.contentItems.push(item));
                this.tabs[this.selectedTab].content = this.contentItems;
            });
        },
        subscribeHandler() {
            if (this.authenticated) {
                this.fetchJson(this.authApiUrl() + (this.subscribed ? "/unsubscribe" : "/subscribe"), null, {
                    method: "POST",
                    body: JSON.stringify({
                        channelId: this.channel.id,
                    }),
                    headers: {
                        Authorization: this.getAuthToken(),
                        "Content-Type": "application/json",
                    },
                });
            } else {
                if (!this.handleLocalSubscriptions(this.channel.id)) return;
            }
            this.subscribed = !this.subscribed;
        },
        getTranslatedTabName(tabName) {
            let translatedTabName = tabName;
            switch (tabName) {
                case "livestreams":
                    translatedTabName = this.$t("titles.livestreams");
                    break;
                case "playlists":
                    translatedTabName = this.$t("titles.playlists");
                    break;
                case "channels":
                    translatedTabName = this.$t("titles.channels");
                    break;
                case "shorts":
                    translatedTabName = this.$t("video.shorts");
                    break;
                default:
                    console.error(`Tab name "${tabName}" is not translated yet!`);
                    break;
            }
            return translatedTabName;
        },
        loadTab(index) {
            this.selectedTab = index;
            if (index == 0) {
                this.contentItems = this.channel.relatedStreams;
                return;
            }
            if (this.tabs[index].content) {
                this.contentItems = this.tabs[index].content;
                return;
            }
            this.fetchJson(this.apiUrl() + "/channels/tabs", {
                data: this.tabs[index].data,
            }).then(tab => {
                this.contentItems = this.tabs[index].content = tab.content;
                this.tabs[this.selectedTab].tabNextPage = tab.nextpage;
            });
        },
    },
};
</script>
