<template>
  <div class="about">
    <h1>CodeMirror Example</h1>
    <codemirror
      v-model="code"
      :style="{ height: '400px' , width: '1000px' }"
      :autofocus="true"
      :indent-with-tab="true"
      :tab-size="2"
      :extensions="extensions"
    />
  </div>
</template>

<script>
  import { basicSetup } from 'codemirror'
  import { defineComponent, ref } from 'vue'
  import { Codemirror } from 'vue-codemirror'
  import { java } from '@codemirror/lang-java'
  import { githubLight } from '@fsegurai/codemirror-theme-github-light';
  import { autocompletion } from '@codemirror/autocomplete'
  import { indentWithTab } from '@codemirror/commands';
  import { keymap } from '@codemirror/view'

  export default defineComponent({
    name: 'AboutView',
    components: {
      Codemirror
    },
    setup() {
      // Sample code to display

      const code = ref('String value = "hello world";\n' +
        'if (value.equals("hello")){\n' +
        ' System.out.println("Hello World");\n' +
        '}\n');

      // First, let's define the parameter suggestions
      const functionParameters = {
        'addGroup': [
          { label: 'uuid', detail: 'String', type: 'parameter' },
          { label: 'grpName', detail: 'String', type: 'parameter' },
          { label: 'propertiesMap', detail: 'Map<String, String>', type: 'parameter' }
        ],
        'setProperties': [
          { label: 'uuid', detail: 'String', type: 'parameter' },
          { label: 'grpName', detail: 'String', type: 'parameter' },
          { label: 'properties', detail: 'Map<String, String>', type: 'parameter' }
        ],
        'removeGroup': [
          { label: 'uuid', detail: 'String', type: 'parameter' },
          { label: 'grpName', detail: 'String', type: 'parameter' }
        ]
      };

      // Define the hierarchical suggestions
      const suggestions = {
        '': [{ label: 'ws', type: 'variable' }],
        'ws': [{ label: 'propertyGroup', type: 'property' }],
        'ws.propertyGroup': [
          { label: 'addGroup', type: 'function' },
          { label: 'setProperties', type: 'function' },
          { label: 'removeGroup', type: 'function' }
        ]
      };

      // Create a function that provides completions based on the context
      function myCompletions(context) {
        const cursor = context.pos;
        const doc = context.state.doc;
        const line = doc.lineAt(cursor);
        const lineText = line.text;

        // Find the nearest opening parenthesis before cursor
        const openParenPos = lineText.lastIndexOf('(', cursor - line.from);
        if (openParenPos !== -1) {
          // We're inside function parameters
          const beforeParen = lineText.slice(0, openParenPos);
          const functionName = beforeParen.split('.').pop();

          // Check if the opening parenthesis belongs to the function
          const isFunctionParen = beforeParen.trim().endsWith(functionName);

          if (isFunctionParen && functionParameters[functionName]) {
            // Count commas before cursor position to determine which parameter to suggest
            const textBeforeCursor = lineText.slice(openParenPos + 1, cursor - line.from);
            const commaCount = (textBeforeCursor.match(/,/g) || []).length;

            // Check if the cursor is before the closing parenthesis
            const afterCursor = lineText.slice(cursor - line.from);
            const closingParenPos = afterCursor.indexOf(')');
            const isCursorBeforeClosingParen = closingParenPos !== -1;

            if (isCursorBeforeClosingParen) {
              const params = functionParameters[functionName];
              if (commaCount < params.length) {
                return {
                  from: cursor,
                  options: [params[commaCount]]
                };
              }
            }
          }
        }

        // Regular completion logic for non-parameter contexts
        const word = context.matchBefore(/[\w.]*/);
        if (!word || (word.from === word.to && !context.explicit)) return null;

        const textBeforeCursor = word.text;

        if (textBeforeCursor.endsWith('.')) {
          const prefix = textBeforeCursor.slice(0, -1);
          return {
            from: word.from + textBeforeCursor.length,
            options: suggestions[prefix] || []
          };
        }

        const parts = textBeforeCursor.split('.');
        const lastPart = parts[parts.length - 1];
        const prefix = parts.slice(0, -1).join('.');

        const options = !prefix ?
          suggestions[''].filter(item => item.label.toLowerCase().startsWith(lastPart.toLowerCase())) :
          (suggestions[prefix] || []).filter(item => item.label.toLowerCase().startsWith(lastPart.toLowerCase()));

        return {
          from: word.from + textBeforeCursor.length - lastPart.length,
          options: options
        };
      }

      // Configure extensions
      const extensions = [
        basicSetup,
        java(),
        githubLight,
        autocompletion({ override: [myCompletions] }),
        keymap.of([indentWithTab]),
      ];

      return {
        code,
        extensions
      }
    }
  });
</script>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    padding: 20px;
  }
}
</style>

