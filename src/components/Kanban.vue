<template>
  <div class="drag-container">
    <ul class="drag-list">
      <li v-for="stage in stages" class="drag-column" :class="{['drag-column-' + stage]: true}" :key="stage">
        <span class="drag-column-header">
          <slot :name="stage">
            <h2>{{ stage }} ABC LEANDRO</h2>
          </slot>
        </span>
        <div class="drag-options"></div>
        <ul class="drag-inner-list" ref="list" :data-status="stage">
          <li class="drag-item" v-for="block in getBlocks(stage)" :data-block-id="block[idProp]" :key="block[idProp]">
            <slot :name="block[idProp]">
              <strong>{{ block[statusProp] }}</strong>
              <div>{{ block[idProp] }}</div>
            </slot>
          </li>
        </ul>
        <div class="drag-column-footer">
            <slot :name="`footer-${stage}`"></slot>
        </div>
      </li>
    </ul>
  </div>
</template>

<script>
  import dragula from 'dragula';
  import { Machine } from 'xstate';

  export default {
    name: 'KanbanBoard',

    props: {
      stages: {
        type: Array,
        required: true,
      },
      blocks: {
        type: Array,
        required: true,
      },
      config: {
        type: Object,
        default: () => ({}),
      },
      stateMachineConfig: {
        type: Object,
        default: null,
      },
      idProp: {
        type: String,
        default: 'id',
      },
      statusProp: {
        type: String,
        default: 'status',
      },
    },

    data() {
      return {
        machine: null,
      };
    },

    computed: {
      localBlocks() {
        return this.blocks;
      },
    },

    methods: {
      getBlocks(status) {
        return this.localBlocks.filter(block => block[this.statusProp] === status);
      },

      findPossibleTransitions(sourceState) {
        return this.machine.config.states[sourceState].on || {};
      },

      findTransition(target, source) {
        const targetState = target.dataset.status;
        const sourceState = source.dataset.status;
        const possibleTransitions = this.findPossibleTransitions(sourceState);
        return Object.keys(possibleTransitions)
          .find(transition => possibleTransitions[transition] === targetState);
      },

      accepts(block, target, source) {
        if (!this.machine) return true;
        const targetState = target.dataset.status;
        const sourceState = source.dataset.status;
        return Object.values(this.findPossibleTransitions(sourceState)).includes(targetState);
      },

      allowedTargets(el, source) {
        const block = this.localBlocks.find(b => b[this.idProp] === el.dataset.blockId);
        return this.drake.containers.filter(c => this.config.accepts(block, c, source));
      },

      forbiddenTargets(el, source) {
        return this.drake.containers.filter(c => !this.allowedTargets(el, source).includes(c));
      },
    },

    updated() {
      this.drake.containers = this.$refs.list;
      this.drake.mirrorContainer = this.$el;
    },

    mounted() {
      this.config.accepts = this.config.accepts || this.accepts;
      this.config.mirrorContainer = this.$el;
      this.drake = dragula(this.$refs.list, this.config)
      .on('drag', (el, source) => {
        this.$emit('drag', el, source);
        el.classList.add('is-moving');
        this.allowedTargets(el, source).forEach(c => c.classList.add('allowed'));
        this.forbiddenTargets(el, source).forEach(c => c.classList.add('forbidden'));
      })
      .on('dragend', (el) => {
        this.$emit('dragend', el);
        el.classList.remove('is-moving');
        this.drake.containers.forEach(c => c.classList.remove('allowed', 'forbidden'));
        window.setTimeout(() => {
          el.classList.add('is-moved');
          window.setTimeout(() => {
            el.classList.remove('is-moved');
          }, 600);
        }, 100);
      })
      .on('drop', (block, list, source, sibling) => {
        this.$emit('drop', block, list, source, sibling);
        let index = 0;
        for (index = 0; index < list.children.length; index += 1) {
          if (list.children[index].classList.contains('is-moving')) break;
        }

        let newState = list.dataset.status;

        if (this.machine) {
          const transition = this.findTransition(list, source);
          if (!transition) return;
          newState = this.machine.transition(source.dataset.status, transition).value;
        }

        this.$emit('update-block', block.dataset.blockId, newState, index);
      })
      .on('cancel', (el, container, source) => {
        this.$emit('cancel', el, container, source);
      })
      .on('remove', (el, container, source) => {
        this.$emit('remove', el, container, source);
      })
      .on('shadow', (el, container, source) => {
        this.$emit('shadow', el, container, source);
      })
      .on('over', (el, container, source) => {
        this.$emit('over', el, container, source);
      })
      .on('out', (el, container, source) => {
        this.$emit('out', el, container, source);
      })
      .on('cloned', (clone, original, type) => {
        this.$emit('cloned', clone, original, type);
      });
      this.$emit('init', this.drake);
    },

    created() {
      if (!this.stateMachineConfig) return;
      this.machine = Machine(this.stateMachineConfig);
    },
  };
</script>
