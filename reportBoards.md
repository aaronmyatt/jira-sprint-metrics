# Report Boards With Table

Fetches all Jira boards by delegating to the boards sub-pipeline and renders them as a sorted Markdown table with ID, Name, and Type columns. If input.selectedBoard is set, an additional Selected column is appended and the matching board is marked with a checkmark. The final Markdown table string is written to input.body.

## Get Boards

```ts
import formatTableAs from "jsr:@dep/table";
import getBoards from "boards";

Object.assign(input, await getBoards.process());

const tableHeaders = ["ID", "Name", "Type"];
if(input.selectedBoard) {
tableHeaders.push("Selected");
}

input.body = new formatTableAs.Markdown()
    .add(...tableHeaders);

input.boards
.toSorted((left, right) => left.name.localeCompare(right.name))
.forEach(board => {
  const columns = [board.id,
    board.name,
    board.type || "unknown",
  ]
  if(input.selectedBoard) {
    columns.push(board.id === input.selectedBoard.id ? "✅" : "");
  }
  input.body.add(...columns)
});    

input.body = input.body.build();
```

