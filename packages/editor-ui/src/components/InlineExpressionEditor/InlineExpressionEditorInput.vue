<template>
	<div ref="root" data-test-id="inline-expression-editor-input"></div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { mapStores } from 'pinia';
import { EditorView, keymap } from '@codemirror/view';
import { Compartment, EditorState, Prec } from '@codemirror/state';
import { history, redo } from '@codemirror/commands';
import { acceptCompletion, autocompletion, completionStatus } from '@codemirror/autocomplete';

import { useNDVStore } from '@/stores/ndv.store';
import { workflowHelpers } from '@/mixins/workflowHelpers';
import { expressionManager } from '@/mixins/expressionManager';
import { highlighter } from '@/plugins/codemirror/resolvableHighlighter';
import { expressionInputHandler } from '@/plugins/codemirror/inputHandlers/expression.inputHandler';
import { inputTheme } from './theme';
import { n8nLang } from '@/plugins/codemirror/n8nLang';
import { completionManager } from '@/mixins/completionManager';

const editableConf = new Compartment();

export default defineComponent({
	name: 'InlineExpressionEditorInput',
	mixins: [completionManager, expressionManager, workflowHelpers],
	props: {
		value: {
			type: String,
		},
		isReadOnly: {
			type: Boolean,
			default: false,
		},
		isSingleLine: {
			type: Boolean,
			default: false,
		},
		path: {
			type: String,
		},
	},
	watch: {
		isReadOnly(newValue: boolean) {
			this.editor?.dispatch({
				effects: editableConf.reconfigure(EditorView.editable.of(!newValue)),
			});
		},
		value(newValue) {
			const isInternalChange = newValue === this.editor?.state.doc.toString();

			if (isInternalChange) return;

			// manual update on external change, e.g. from expression modal or mapping drop

			this.editor?.dispatch({
				changes: {
					from: 0,
					to: this.editor?.state.doc.length,
					insert: newValue,
				},
			});
		},
		ndvInputData() {
			this.editor?.dispatch({
				changes: {
					from: 0,
					to: this.editor.state.doc.length,
					insert: this.value,
				},
			});

			setTimeout(() => {
				this.editor?.contentDOM.blur();
			});
		},
	},
	computed: {
		...mapStores(useNDVStore),
		ndvInputData(): object {
			return this.ndvStore.ndvInputData;
		},
	},
	mounted() {
		const extensions = [
			inputTheme({ isSingleLine: this.isSingleLine }),
			Prec.highest(
				keymap.of([
					{ key: 'Tab', run: acceptCompletion },
					{
						any(view: EditorView, event: KeyboardEvent) {
							if (event.key === 'Escape' && completionStatus(view.state) !== null) {
								event.stopPropagation();
							}

							return false;
						},
					},
					{ key: 'Mod-Shift-z', run: redo },
				]),
			),
			autocompletion(),
			n8nLang(),
			history(),
			expressionInputHandler(),
			EditorView.lineWrapping,
			editableConf.of(EditorView.editable.of(!this.isReadOnly)),
			EditorView.contentAttributes.of({ 'data-gramm': 'false' }), // disable grammarly
			EditorView.domEventHandlers({
				focus: () => {
					this.$emit('focus');
				},
			}),
			EditorView.updateListener.of((viewUpdate) => {
				if (!this.editor || !viewUpdate.docChanged) return;

				highlighter.removeColor(this.editor, this.plaintextSegments);
				highlighter.addColor(this.editor, this.resolvableSegments);

				setTimeout(() => {
					try {
						this.trackCompletion(viewUpdate, this.path);
					} catch {}
				});

				this.$emit('change', {
					value: this.unresolvedExpression,
					segments: this.displayableSegments,
				});
			}),
		];

		this.editor = new EditorView({
			parent: this.$refs.root as HTMLDivElement,
			state: EditorState.create({
				doc: this.value.startsWith('=') ? this.value.slice(1) : this.value,
				extensions,
			}),
		});

		highlighter.addColor(this.editor, this.resolvableSegments);

		this.$emit('change', {
			value: this.unresolvedExpression,
			segments: this.displayableSegments,
		});
	},
	destroyed() {
		this.editor?.destroy();
	},
	methods: {
		focus() {
			this.editor?.focus();
		},
	},
});
</script>

<style lang="scss"></style>
