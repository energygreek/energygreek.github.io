---
draft: true
comments: true
date: 2025-05-20
tags: [cli,typescript]
---

# implement cli use typescript  
i have heard that typescript has a lot of packages already, so i tried it with guide of chatgpt.

## steps of a todo command line cli tool

mkdir todo-cli && cd todo-cli  

npm init -y                       
npm install inquirer chalk     
- inquirer: For interactive terminal prompts.
- chalk: For colored terminal output.  

npm install --save-dev typescript ts-node @types/node @types/inquirer
- typescript: TypeScript compiler.
- ts-node: Run .ts files directly.
- @types/...: Type definitions for Node.js and Inquirer (required for TypeScript support).


npx tsc --init       
- Generates a default TypeScript config file (tsconfig.json), which controls how TS is compiled.

vi todo.ts            
```ts
import fs from 'fs';
import inquirer from 'inquirer';
import chalk from 'chalk';

const FILE = 'todos.json';

type Todo = {
  text: string;
  done: boolean;
};

function loadTodos(): Todo[] {
  if (!fs.existsSync(FILE)) return [];
  return JSON.parse(fs.readFileSync(FILE, 'utf-8'));
}

function saveTodos(todos: Todo[]) {
  fs.writeFileSync(FILE, JSON.stringify(todos, null, 2));
}

async function main() {
  const todos = loadTodos();

  const { action } = await inquirer.prompt({
    type: 'list',
    name: 'action',
    message: 'Choose action:',
    choices: ['List TODOs', 'Add TODO', 'Mark TODO as done', 'Exit'],
  });

  switch (action) {
    case 'List TODOs':
      if (todos.length === 0) {
        console.log(chalk.gray('No TODOs found.'));
      } else {
        todos.forEach((todo, i) => {
          console.log(`${todo.done ? chalk.green('[x]') : chalk.red('[ ]')} ${i + 1}. ${todo.text}`);
        });
      }
      break;

    case 'Add TODO':
      const { text } = await inquirer.prompt({
        type: 'input',
        name: 'text',
        message: 'Enter TODO text:',
      });
      todos.push({ text, done: false });
      saveTodos(todos);
      console.log(chalk.green('TODO added.'));
      break;

    case 'Mark TODO as done':
      if (todos.length === 0) {
        console.log(chalk.gray('No TODOs to mark.'));
        break;
      }
      const { index } = await inquirer.prompt({
        type: 'list',
        name: 'index',
        message: 'Select TODO to mark as done:',
        choices: todos.map((t, i) => ({
          name: `${t.done ? '[x]' : '[ ]'} ${t.text}`,
          value: i,
        })),
      });
      todos[index].done = true;
      saveTodos(todos);
      console.log(chalk.green('Marked as done.'));
      break;
  }

  if (action !== 'Exit') main();
}

main();
```

npx ts-node todo.ts      