<template>
    <div class="uk-container uk-container-xlarge" v-if="video">
        <ErrorHandler v-if="video && video.error" :message="video.message" :error="video.error" />

        <div v-show="!video.error">
            <div v-if="this.isPlaylist">
                <div class="uk-grid uk-grid-match" uk-grid>
                    <div class="uk-width-expand@m">
                        <Player
                            ref="videoPlayer"
                            :video="video"
                            :sponsors="sponsors"
                            :selectedAutoPlay="selectedAutoPlay"
                            :selectedAutoLoop="selectedAutoLoop"
                            :autoPlaylist="isPlaylist"
                            :playlist="playlist"
                        />
                    </div>
                    <div style="overflow-y: scroll; height: 100%; max-height: 75vh; min-height: 250px;" class="uk-width-1-4@m uk-height-max-large">
                        <div
                            v-bind:key="video.url"
                            v-for="(video, index) in this.playlist.relatedStreams"
                        >
                            <VideoItem :playlistIndex="index" :isPlaylist="isPlaylist" :video="video" height="60" width="60" />
                        </div>
                    </div>
                </div>
            </div>
            <div v-else-if="!this.isPlaylist">
                <Player
                    ref="videoPlayer"
                    :video="video"
                    :sponsors="sponsors"
                    :selectedAutoPlay="selectedAutoPlay"
                    :selectedAutoLoop="selectedAutoLoop"
                />
            </div>

            <div class="uk-text-bold uk-margin-small-top uk-text-large uk-text-emphasis">{{ video.title }}</div>

            <div class="uk-flex uk-flex-middle">
                <div class="uk-margin-small-right">{{ addCommas(video.views) }} views</div>
                <div class="uk-margin-small-right">{{ video.uploadDate }}</div>
                <div class="uk-flex-1"></div>
                <div class="uk-margin-small-left">
                    <font-awesome-icon class="uk-margin-small-right" icon="thumbs-up"></font-awesome-icon>
                    <b>{{ addCommas(video.likes) }}</b>
                </div>
                <div class="uk-margin-small-left">
                    <font-awesome-icon class="uk-margin-small-right" icon="thumbs-down"></font-awesome-icon>
                    <b>{{ addCommas(video.dislikes) }}</b>
                </div>
                <a
                    :href="'https://youtu.be/' + getVideoId()"
                    class="uk-margin-small-left uk-button uk-button-small"
                    style="background: #222"
                >
                    <font-awesome-icon class="uk-margin-small-right" :icon="['fab', 'youtube']"></font-awesome-icon>
                    <b>Watch on</b>
                </a>
            </div>

            <div class="uk-flex uk-flex-middle uk-margin-small-top">
                <img :src="video.uploaderAvatar" loading="lazy" />
                <router-link class="uk-text-bold uk-margin-small-left" v-if="video.uploaderUrl" :to="video.uploaderUrl">
                    <a>{{ video.uploader }}</a>
                </router-link>
                <div class="uk-flex-1"></div>
                <button
                    v-if="authenticated"
                    @click="subscribeHandler"
                    class="uk-button uk-button-small"
                    style="background: #222"
                    type="button"
                >
                    {{ subscribed ? "Unsubscribe" : "Subscribe" }}
                </button>
            </div>

            <hr />

            <a class="uk-button uk-button-small" style="background: #222" @click="showDesc = !showDesc">
                {{ showDesc ? "Minimize Description" : "Show Description" }}
            </a>
            <p v-show="showDesc" :style="[{ colour: foregroundColor }]" v-html="video.description"></p>
            <div v-if="showDesc && sponsors && sponsors.segments">
                Sponsors Segments: {{ sponsors.segments.length }}
            </div>
        </div>

        <hr />

        <b>Loop this Video:</b>&nbsp;
        <input class="uk-checkbox" v-model="selectedAutoLoop" @change="onChange($event)" type="checkbox" />
        <br />
        <b>Auto Play next Video:</b>&nbsp;
        <input class="uk-checkbox" v-model="selectedAutoPlay" @change="onChange($event)" type="checkbox" />

        <hr />

        <div uk-grid>
            <div class="uk-width-4-5@xl uk-width-3-4@l uk-width-1" v-if="comments" ref="comments">
                <div
                    class="uk-tile-default uk-align-left uk-width-expand"
                    :style="[{ background: backgroundColor }]"
                    v-bind:key="comment.commentId"
                    v-for="comment in comments.comments"
                >
                    <Comment :comment="comment" :uploader="video.uploader" />
                </div>
            </div>

            <div class="uk-width-1-5@xl uk-width-1-4@l uk-width-1 uk-flex-last@l uk-flex-first" v-if="video">
                <div
                    class="uk-tile-default uk-width-auto"
                    :style="[{ background: backgroundColor }]"
                    v-bind:key="related.url"
                    v-for="related in video.relatedStreams"
                >
                    <VideoItem :video="related" height="94" width="168" />
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import Player from "@/components/Player.vue";
import VideoItem from "@/components/VideoItem.vue";
import ErrorHandler from "@/components/ErrorHandler.vue";
import Comment from "@/components/Comment.vue";

export default {
    name: "App",
    data() {
        return {
            video: {
                title: "Loading...",
            },
            sponsors: null,
            selectedAutoLoop: false,
            selectedAutoPlay: null,
            showDesc: true,
            comments: null,
            subscribed: false,
            channelId: null,
            active: true,
            isPlaylist: false,
            playlist: {},
            currentPlaylistIndex: undefined,
        };
    },
    mounted() {
        this.getVideoData().then(() => {
            if (this.active) this.$refs.videoPlayer.loadVideo();
        });
        this.getSponsors();
        if (this.getPlaylistId()) this.isPlaylist = true;
        if (this.isPlaylist) this.getPlaylistData()
        if (this.getPreferenceBoolean("comments", true)) this.getComments();
    },
    activated() {
        this.active = true;
        this.selectedAutoPlay = this.getPreferenceBoolean("autoplay", true);
        if (this.video.duration) {
            document.title = this.video.title + " - Piped";
            this.$refs.videoPlayer.loadVideo();
        }
        window.addEventListener("scroll", this.handleScroll);
    },
    deactivated() {
        this.active = false;
        window.removeEventListener("scroll", this.handleScroll);
    },
    watch: {
        "$route.query.v": function(v) {
            if (v) {
                window.scrollTo(0, 0);
            }
        },
    },
    methods: {
        fetchVideo() {
            return this.fetchJson(this.apiUrl() + "/streams/" + this.getVideoId());
        },
        async fetchSponsors() {
            return await this.fetchJson(this.apiUrl() + "/sponsors/" + this.getVideoId(), {
                category:
                    '["' +
                    this.getPreferenceString("selectedSkip", "sponsor,interaction,selfpromo,music_offtopic").replaceAll(
                        ",",
                        '","',
                    ) +
                    '"]',
            });
        },
        fetchComments() {
            return this.fetchJson(this.apiUrl() + "/comments/" + this.getVideoId());
        },
        onChange() {
            this.setPreference("autoplay", this.selectedAutoPlay);
        },
        async getVideoData() {
            await this.fetchVideo()
                .then(data => {
                    this.video = data;
                })
                .then(() => {
                    if (!this.video.error) {
                        document.title = this.video.title + " - Piped";
                        this.channelId = this.video.uploaderUrl.split("/")[2];
                        this.fetchSubscribedStatus();

                        this.video.description = this.purifyHTML(
                            this.video.description
                                .replaceAll("http://www.youtube.com", "")
                                .replaceAll("https://www.youtube.com", "")
                                .replaceAll("\n", "<br>"),
                        );
                    }
                });
        },
        async getSponsors() {
            if (this.getPreferenceBoolean("sponsorblock", true))
                this.fetchSponsors().then(data => (this.sponsors = data));
        },
        async getComments() {
            this.fetchComments().then(data => (this.comments = data));
        },
        async fetchSubscribedStatus() {
            if (!this.channelId || !this.authenticated) return;

            this.fetchJson(
                this.apiUrl() + "/subscribed",
                {
                    channelId: this.channelId,
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
        subscribeHandler() {
            this.fetchJson(this.apiUrl() + (this.subscribed ? "/unsubscribe" : "/subscribe"), null, {
                method: "POST",
                body: JSON.stringify({
                    channelId: this.channelId,
                }),
                headers: {
                    Authorization: this.getAuthToken(),
                    "Content-Type": "application/json",
                },
            });
            this.subscribed = !this.subscribed;
        },
        handleScroll() {
            if (this.loading || !this.comments || !this.comments.nextpage) return;
            if (window.innerHeight + window.scrollY >= this.$refs.comments.offsetHeight - window.innerHeight) {
                this.loading = true;
                this.fetchJson(this.apiUrl() + "/nextpage/comments/" + this.getVideoId(), {
                    url: this.comments.nextpage,
                }).then(json => {
                    this.comments.nextpage = json.nextpage;
                    this.loading = false;
                    json.comments.map(comment => this.comments.comments.push(comment));
                });
            }
        },
        getVideoId() {
            return this.$route.query.v || this.$route.params.v;
        },
        getPlaylistId() {
            return this.$route.query.list || this.$route.params.list;
        },
        getIndex() {
            return this.$route.query.index || undefined;
        },
        async fetchPlaylist() {
            return await this.fetchJson(`${this.apiUrl()}/playlists/${this.getPlaylistId()}`)
        },
        async getPlaylistData() {
            this.fetchPlaylist()
                .then(data => {
                    this.playlist = data
                }).then(() => {
                    const currentIndex = this.playlist.relatedStreams.findIndex(v => v.url.includes(this.getVideoId()));
                    if (!this.getIndex()) {
                        console.log(currentIndex, 'here')
                        this.$router.push(`${this.$route.fullPath}&index=${currentIndex}`)
                    }
                })
        },
    },
    components: {
        Player,
        VideoItem,
        ErrorHandler,
        Comment,
    },
};
</script>
