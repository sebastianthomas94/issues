## Problem Statement

We have a rich repository of student and system-generated data with strong business value. However, our current UI and REST APIs are static and cannot meet the dynamic needs of business users.

Business teams often need flexible ways to filter, group, and visualize data, but our tools limit them.

### Example requests we can’t easily support:

- Students who joined last month via referrals
- Weekly student sign-up trends (bar chart)
- Livestreams scheduled for tomorrow

This lack of flexibility slows insights and decision-making.

## Proposed Solution

We can leverage GraphQL and LLMs to create dynamic, user-driven dashboards.

1. **GraphQL Module with RBAC** – Build a secure GraphQL layer that exposes all relevant data while enforcing role-based access control.
2. **Natural Language to Query** – Users provide requests in plain language, which are processed by an LLM to generate appropriate GraphQL queries.
3. **Dynamic UI Generation** – The LLM also returns a JSON structure describing how to visualize the results (e.g., tables, charts, graphs). This is rendered on the Next.js client.
4. **Save and Reuse** – Users can review and save these custom queries and UI layouts for later use. When revisited, the saved UI fetches fresh data automatically.

This approach enables flexible, self-service access to data without requiring constant backend changes.

## Implementation Challenges

1. **Large GraphQL Schema** – Sending the entire schema to the LLM is costly and can cause errors. We need a parser to extract only relevant parts for each query.

2. **Dynamic UI** – Raw HTML can’t be rendered in Next.js. We need server-driven UI (JSON blocks) that the client interprets into components.

Here’s a more concise and polished version:

## Key Points for Implementation

1. **Reusable LLM Module** – Design the LLM component to be universal and reusable across other modules.
2. **Query Matching & Caching** – Implement caching and semantic matching to avoid repeated or duplicate queries for the same data.

# ⚒️Implementation

### SDUI components

```ts
type UIType: "pi-chart" | "bar-chart" | "line-chart" | "list-view";

class MainUIComponent{
    id: string;
    type: UIType;
    query: string;  // gql query
    title?:string;
    caption?:string;
}

class PieChart extends UIComponent {
  type: "pi-chart";
  data: {
    label: string;
    value: number;  // should be converted to percentile of pi in the client
  }[];
}

class BarChart extends UIComponent {
  type: "bar-chart";
  xAxis: string[];
  series: {
    label: string;
    yAxis: number[];
  }[]; // array for multiple bars in same graph
}

class LineChart extends UIComponent {
  type: "line-chart";
  xAxis: string[];
  series: {
    label: string;
    yAxis: number[];
  }[]; // array for multiple bars in same graph
}

class ListView extends UIComponent {
    type: "list-view";
    headers?: string[]; // if headers exist it can be displayed as a table
    items: any[];
}

type Component = PieChart | BarChart | LineChart | ListView;

class Screen {
    components: Component[];
}

```

### **Route**

**POST** `/reports`

**Request Body:**

```json
{
  "prompt": "string"
}
```

**Response:**

```json
{
  "data": "Screen"
}
```

**GET** `/reports`

```ts
class GetReportsDto {
  reports: { id: string; tag: string }[];
}
```

**GET** `/reports`

**Request Body:**

```json
{
  "id": "string"
}
```

```ts
class GetReportScreenDto {
  data: Screen;
}
```
