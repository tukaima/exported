<template>
    <span :class="linkClasses" v-if="character">
        <slot v-if="deleted">[Deleted] {{ name }}</slot>
        <a :href="characterUrl" class="characterLinkLink" v-else><slot>{{ name }}</slot></a>
    </span>
</template>

<script lang="ts">
    import Vue from 'vue';
    import Component from 'vue-class-component';
    import {Prop} from 'vue-property-decorator';
    import * as Utils from '../site/utils';

    @Component
    export default class CharacterLink extends Vue {
        @Prop({required: true})
        readonly character!: {name: string, id: number, deleted: boolean} | string;

        get deleted(): boolean {
            return typeof(this.character) === 'string' ? false : this.character.deleted;
        }

        get linkClasses(): string {
            return this.deleted ? 'characterLinkDeleted' : 'characterLink';
        }

        get characterUrl(): string {
            return Utils.characterURL(this.name);
        }

        get name(): string {
            return typeof(this.character) === 'string' ? this.character : this.character.name;
        }
    }
</script>