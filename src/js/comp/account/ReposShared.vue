<!--
    Copyright (c) 2016, German Neuroinformatics Node (G-Node)

    All rights reserved.

    Redistribution and use in source and binary forms, with or without
    modification, are permitted under the terms of the BSD License. See
    LICENSE file in the root of the Project.
-->

<template>
    <div>
        <!-- pagination -->
        <nav v-if="repos_modified">
            <ul class="pager">
                <li class="previous"><a @click="prev">
                    Previous <span v-if="pidx !== null">({{ pidx }})</span>
                </a></li>
                <li class="next"><a @click="next">
                    Next <span v-if="nidx !== null">({{ nidx }})</span>
                </a></li>
            </ul>
        </nav>

        <ul class="list-unstyled" v-if="repos_modified">
            <li v-for="repo in repos_displayed">
                <div class="panel panel-default">
                    <div class="panel-heading">
                        <router-link :to="{ name: 'repository-info',
                                            params: { username: repo.Owner, repository: repo.Name } }">
                            {{ repo.Owner }}/{{ repo.Name }}
                        </router-link>

                        <strong v-if="repo.Public"><span class="label label-success pull-right">public</span></strong>
                        <strong v-if="!repo.Public"><span class="label label-primary pull-right">private</span></strong>
                    </div>
                    <div class="panel-body">
                        Owner {{ repo.FullName }}<br/><br/>
                        {{ repo.Description }}
                        <span v-if="!repo.Description">No description for this repository.</span>
                    </div>
                </div>
            </li>
        </ul>
        <ul class="list-unstyled" v-if="!repos_modified">
            <li class="panel panel-default">
                <div class="panel-body">
                    There are no available repositories
                </div>
            </li>
        </ul>
    </div>
</template>

<script type="text/ecmascript-6">
    import Alert from "../Alert.js"
    import {
            pagerPrevious,
            pagerNext,
            addRepoUserFullName
    } from "../../utils.js"

    const ll = "[ReposShared]"
    const n_displayed = 5

    export default {
        data() {
            return {
                repos_modified: null,
                repos_displayed: null,
                idx: 0,
                pidx: null,
                nidx: null
            }
        },

        mounted() {
            this.update(this.$route.params)
        },

        props: {
            account: { required: true }
        },

        mixins: [Alert],

        methods: {

            update(params) {
                window.log.print("Debug", `${ll} update`)
                this.repos_modified = null
                this.repos_displayed = null

                const promise = window.api.repos.listShared()
                promise.then(
                        (repos) => {
                            if (repos !== undefined && repos !== null && repos.length > 0) {
                                var repo_list = repos

                                // A logged in user can pre-filter repositories shared
                                // by another repository owner via the route.
                                if (this.account.login !== params.username) {
                                    const repo_filter = Array.from(repos)
                                            .filter((r) => { return r.Owner === params.username })
                                    if (repo_filter.length > 0) {
                                        repo_list = repo_filter
                                    } else {
                                        return
                                    }
                                }

                                // Update repository list with full name of the repository owners.
                                const user_search = window.api.accounts.search()
                                user_search.then(
                                        (u) => {
                                            this.repos_modified = addRepoUserFullName(u, repo_list)

                                            this.idx = 0
                                            const len = n_displayed > this.repos_modified.length ?
                                                    this.repos_modified.length : n_displayed
                                            this.repos_displayed = this.repos_modified.slice(this.idx, len)

                                            this.updatePagerIndex()
                                        },
                                        (error) => {
                                            window.log.print("Err", `${ll} error fetching user for shared repos`)
                                            window.log.print("Err", error)
                                        })
                            }
                        },
                        (error) => {
                            window.log.print("Err", `${ll} error fetching shared repos`)
                            window.log.print("Err", error)
                        })
            },

            prev() {
                const prev = pagerPrevious(this.repos_modified, this.idx, n_displayed)
                this.repos_displayed = prev.arr
                this.idx = prev.index
                this.updatePagerIndex()
            },

            next() {
                const nxt = pagerNext(this.repos_modified, this.idx, n_displayed)
                this.repos_displayed = nxt.arr
                this.idx = nxt.index
                this.updatePagerIndex()
            },

            updatePagerIndex() {
                this.pidx = Math.floor(this.idx / n_displayed)
                this.nidx = Math.floor((this.repos_modified.length - this.idx) / n_displayed)
                if ((this.repos_modified.length - this.idx) === n_displayed) {
                    this.nidx = 0
                }
            }
        },

        watch: {
            "$route.params": function(params) {
                this.update(params)
            }
        }
    }
</script>
