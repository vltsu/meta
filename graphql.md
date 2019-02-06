> We recommend having one schema that describes your entire data universe.

https://github.com/facebook/relay/issues/130#issuecomment-133078797

---

- [artsy/metaphysics](https://github.com/artsy/metaphysics) - proxy of REST APIs with schema stitching for inspiration
- https://github.com/artsy/README/blob/master/playbooks/graphql-schema-design.md

# GraphQL errors

See: http://artsy.github.io/blog/2018/10/19/where-art-thou-my-error/

```typescript
import { OrderStatus_order } from "__generated__/OrderStatus_order.graphql"
import React from "react"
import { createFragmentContainer, graphql } from "react-relay"

interface Props {
  order: OrderStatus_order
}

const OrderStatus: React.SFC<Props> = ({ order: orderStatusOrError }) =>
  orderStatusOrError.__typename === "OrderStatus" ? (
    <div>
      {orderStatusOrError.deliveryDispatched
        ? "Your order has been dispatched."
        : "Your order has not been dispatched yet."}
    </div>
  ) : (
    <div className="error">
      {orderStatusOrError.code === "unpublished"
        ? "Please contact gallery services."
        : `An unexpected error occurred: ${orderStatusOrError.message}`}
    </div>
  )

export const OrderStatusContainer = createFragmentContainer(
  OrderStatus,
  graphql`
    fragment OrderStatus_order on Order {
      orderStatusOrError {
        __typename
        ... on OrderStatus {
          deliveryDispatched
        }
        ... on OrderError {
          message
          code
        }
      }
    }
  `
)
```

# Recursive queries

> Take Reddit as an example since it's close to this hypothetical nested comments example. They don't actually query to an unknown depth when fetching nested comments. Instead, they eventually bottom out with a "show more comments" link which can trigger a new query fetch. The technique I illustrated in a prior comment allows for this maximum depth control.

[source](https://github.com/facebook/graphql/issues/91#issuecomment-254895093)

```graphql
{
  messages {
    ...CommentsRecursive
  }
}

fragment CommentsRecursive on Message {
  comments {
    ...CommentFields
    comments {
      ...CommentFields
      comments {
        ...CommentFields
        comments {
          ...CommentFields
          comments {
            ...CommentFields
            comments {
              ...CommentFields
              comments {
                ...CommentFields
                comments {
                  ...CommentFields
                }
              }
            }
          }
        }
      }
    }
  }
}

fragment CommentFields on Comment {
  id
  content
}
```