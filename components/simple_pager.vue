<template>
    <div class="d-flex w-100 my-2 justify-content-between">
        <div>
            <slot name="previous" v-if="!routed">
                <a class="btn btn-secondary" :class="{'disabled': !prev}" role="button" @click.prevent="previousPage">
                    <span aria-hidden="true">&larr;</span> {{prevLabel}}
                </a>
            </slot>
            <router-link v-if="routed" :to="prevRoute" class="btn btn-secondary" :class="{'disabled': !prev}" role="button">
                <span aria-hidden="true">&larr;</span> {{prevLabel}}
            </router-link>
        </div>
        <div>
            <slot name="next" v-if="!routed">
                <a class="btn btn-secondary" :class="{'disabled': !next}" role="button" @click.prevent="nextPage">
                    {{nextLabel}} <span aria-hidden="true">&rarr;</span>
                </a>
            </slot>
            <router-link v-if="routed" :to="nextRoute" class="btn btn-secondary" :class="{'disabled': !next}" role="button">
                {{nextLabel}} <span aria-hidden="true">&rarr;</span>
            </router-link>
        </div>
    </div>
</template>

<script lang="ts">
    import cloneDeep = require('lodash/cloneDeep'); //tslint:disable-line:no-require-imports
    import Vue from 'vue';
    import Component from 'vue-class-component';
    import {Prop} from 'vue-property-decorator';

    type ParamDictionary = {[key: string]: number | undefined};
    interface RouteParams {
        name?: string
        params?: ParamDictionary
    }

    @Component
    export default class SimplePager extends Vue {
        @Prop({default: 'Next Page'})
        readonly nextLabel!: string;
        @Prop({default: 'Previous Page'})
        readonly prevLabel!: string;
        @Prop({required: true})
        readonly next!: boolean;
        @Prop({required: true})
        readonly prev!: boolean;
        @Prop({default: false})
        readonly routed!: boolean;
        @Prop({default: () => ({})})
        readonly route!: RouteParams;
        @Prop({default: 'page'})
        readonly paramName!: string;

        nextPage(): void {
            if(!this.next)
                return;
            this.$emit('next');
        }

        previousPage(): void {
            if(!this.prev)
                return;
            this.$emit('prev');
        }

        get prevRoute(): RouteParams {
            if(this.route.params !== undefined && this.route.params[this.paramName] !== undefined) {
                const newPage = this.route.params[this.paramName]! - 1;
                const clone = cloneDeep(this.route);
                clone.params![this.paramName] = newPage;
                return clone;
            }
            return {};
        }

        get nextRoute(): RouteParams {
            if(this.route.params !== undefined && this.route.params[this.paramName] !== undefined) {
                const newPage = this.route.params[this.paramName]! + 1;
                const clone = cloneDeep(this.route);
                clone.params![this.paramName] = newPage;
                return clone;
            }
            return {};
        }
    }
</script>