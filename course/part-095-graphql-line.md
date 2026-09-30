# Part 95: GraphQL สำหรับ LINE Bot Backend

## บทนำ

GraphQL เป็น query language สำหรับ API ที่พัฒนาโดย Facebook ซึ่งให้ความยืดหยุ่นสูงกว่า REST API โดยเฉพาะเมื่อนำมาใช้กับ LIFF frontend ที่ต้องการข้อมูลหลากหลายรูปแบบ ในบทนี้เราจะสร้างระบบ GraphQL API สำหรับ LINE Bot อย่างสมบูรณ์

---

## 1. ทำไมต้องใช้ GraphQL กับ LINE Systems?

### 1.1 ปัญหาของ REST API กับ LINE Bot

**Over-fetching:** LIFF ต้องการแค่ชื่อและสถานะ แต่ API ส่งข้อมูลทั้งหมด:
```
GET /api/orders/123
Response: { id, userId, lineUserId, items[10], totalAmount, 
            address, paymentMethod, shippingProvider, 
            trackingNumber, createdAt, updatedAt, ... }
```

**Under-fetching:** ต้องเรียก API หลาย calls:
```
GET /api/user/profile
GET /api/user/orders
GET /api/user/loyalty-points
GET /api/user/coupons
```

**GraphQL Solution:**
```graphql
query GetUserDashboard($userId: ID!) {
  user(id: $userId) {
    displayName
    pictureUrl
    loyaltyPoints
    orders(limit: 5) {
      id
      status
      totalAmount
    }
    activeCoupons {
      code
      discount
      expiresAt
    }
  }
}
```

### 1.2 ประโยชน์ของ GraphQL สำหรับ LINE Ecosystem

- **LIFF Integration** - ดึงข้อมูลที่ต้องการได้พอดี
- **Real-time Subscriptions** - รับ notification เมื่อ order status เปลี่ยน
- **Strong typing** - TypeScript integration ดีกว่า REST
- **Apollo DevTools** - Debug ง่าย
- **Schema Documentation** - Auto-generated docs

---

## 2. Schema Design สำหรับ LINE Data

### 2.1 Core Types

```graphql
# schema/types/user.graphql

"""
LINE User ที่ follow บัญชีของเรา
"""
type LineUser {
  id: ID!
  lineUserId: String!
  displayName: String!
  pictureUrl: String
  statusMessage: String
  language: String
  
  # Business data
  email: String
  phone: String
  loyaltyPoints: Int!
  tier: UserTier!
  
  # Relationships
  orders(
    status: OrderStatus
    limit: Int = 10
    offset: Int = 0
  ): [Order!]!
  
  activeCoupons: [Coupon!]!
  cart: Cart
  addresses: [Address!]!
  
  # Meta
  followedAt: DateTime!
  lastActiveAt: DateTime
  isBlocked: Boolean!
  tags: [String!]!
}

enum UserTier {
  BRONZE
  SILVER
  GOLD
  PLATINUM
}

type Address {
  id: ID!
  label: String!
  recipientName: String!
  phone: String!
  street: String!
  subdistrict: String!
  district: String!
  province: String!
  postalCode: String!
  isDefault: Boolean!
}
```

```graphql
# schema/types/order.graphql

type Order {
  id: ID!
  orderNumber: String!
  user: LineUser!
  items: [OrderItem!]!
  
  # Pricing
  subtotal: Float!
  shippingFee: Float!
  discount: Float!
  totalAmount: Float!
  
  # Status
  status: OrderStatus!
  paymentStatus: PaymentStatus!
  
  # Shipping
  shippingAddress: Address
  trackingNumber: String
  estimatedDelivery: DateTime
  
  # Payment
  payment: Payment
  
  # Meta
  notes: String
  createdAt: DateTime!
  updatedAt: DateTime!
  
  # Computed fields
  canCancel: Boolean!
  canReorder: Boolean!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PAID
  PREPARING
  SHIPPED
  DELIVERED
  CANCELLED
  REFUNDED
}

enum PaymentStatus {
  UNPAID
  PENDING
  PAID
  FAILED
  REFUNDED
}

type OrderItem {
  id: ID!
  product: Product!
  quantity: Int!
  unitPrice: Float!
  totalPrice: Float!
  notes: String
}

type Payment {
  id: ID!
  method: PaymentMethod!
  amount: Float!
  status: PaymentStatus!
  transactionId: String
  paidAt: DateTime
}

enum PaymentMethod {
  LINE_PAY
  CREDIT_CARD
  BANK_TRANSFER
  CASH_ON_DELIVERY
  TRUEMONEY
}
```

```graphql
# schema/types/product.graphql

type Product {
  id: ID!
  sku: String!
  name: String!
  description: String
  category: Category!
  
  # Pricing
  price: Float!
  compareAtPrice: Float
  
  # Media
  images: [ProductImage!]!
  primaryImage: ProductImage
  
  # Inventory
  stockQuantity: Int!
  isAvailable: Boolean!
  
  # Attributes
  variants: [ProductVariant!]!
  tags: [String!]!
  
  # Reviews
  averageRating: Float
  reviewCount: Int!
  
  createdAt: DateTime!
  updatedAt: DateTime!
}

type ProductImage {
  url: String!
  altText: String
  sortOrder: Int!
}

type ProductVariant {
  id: ID!
  name: String!
  value: String!
  priceModifier: Float!
  stockQuantity: Int!
}

type Category {
  id: ID!
  name: String!
  slug: String!
  parentCategory: Category
  products(limit: Int = 20): [Product!]!
}
```

### 2.2 Query Types

```graphql
# schema/query.graphql

type Query {
  # User queries
  me: LineUser
  user(id: ID!): LineUser
  users(
    tier: UserTier
    search: String
    limit: Int = 20
    offset: Int = 0
  ): UserConnection!
  
  # Order queries
  order(id: ID!): Order
  orders(
    userId: ID
    status: OrderStatus
    dateFrom: DateTime
    dateTo: DateTime
    limit: Int = 20
    offset: Int = 0
  ): OrderConnection!
  
  # Product queries
  product(id: ID!): Product
  products(
    categoryId: ID
    search: String
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
    limit: Int = 20
    offset: Int = 0
    sortBy: ProductSortField = CREATED_AT
    sortOrder: SortOrder = DESC
  ): ProductConnection!
  
  # Analytics
  analytics(
    dateFrom: DateTime!
    dateTo: DateTime!
  ): AnalyticsSummary!
  
  # LINE specific
  lineUserProfile(lineUserId: String!): LineUserProfile
  richMenus: [RichMenu!]!
}

# Pagination types
type UserConnection {
  nodes: [LineUser!]!
  totalCount: Int!
  pageInfo: PageInfo!
}

type OrderConnection {
  nodes: [Order!]!
  totalCount: Int!
  pageInfo: PageInfo!
}

type ProductConnection {
  nodes: [Product!]!
  totalCount: Int!
  pageInfo: PageInfo!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

enum ProductSortField {
  NAME
  PRICE
  CREATED_AT
  POPULARITY
  RATING
}

enum SortOrder {
  ASC
  DESC
}
```

### 2.3 Mutation Types

```graphql
# schema/mutation.graphql

type Mutation {
  # Order mutations
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(id: ID!, reason: String): CancelOrderPayload!
  reorder(orderId: ID!): CreateOrderPayload!
  
  # Cart mutations
  addToCart(input: AddToCartInput!): Cart!
  updateCartItem(id: ID!, quantity: Int!): Cart!
  removeFromCart(id: ID!): Cart!
  clearCart: Cart!
  
  # User mutations
  updateProfile(input: UpdateProfileInput!): LineUser!
  addAddress(input: AddAddressInput!): Address!
  updateAddress(id: ID!, input: UpdateAddressInput!): Address!
  deleteAddress(id: ID!): Boolean!
  setDefaultAddress(id: ID!): Address!
  
  # LINE specific
  updateRichMenu(id: ID!, input: RichMenuInput!): RichMenu!
  sendMessage(input: SendMessageInput!): SendMessagePayload!
  broadcastMessage(input: BroadcastInput!): BroadcastPayload!
  
  # Review mutations
  createReview(input: CreateReviewInput!): Review!
  
  # Coupon
  redeemCoupon(code: String!): RedeemCouponPayload!
}

# Input types
input CreateOrderInput {
  items: [OrderItemInput!]!
  shippingAddressId: ID!
  paymentMethod: PaymentMethod!
  couponCode: String
  notes: String
}

input OrderItemInput {
  productId: ID!
  variantId: ID
  quantity: Int!
  notes: String
}

input UpdateProfileInput {
  email: String
  phone: String
  birthDate: Date
  gender: Gender
}

input AddToCartInput {
  productId: ID!
  variantId: ID
  quantity: Int!
}

input SendMessageInput {
  lineUserId: String!
  messages: [MessageInput!]!
  type: MessageSendType!
}

enum MessageSendType {
  PUSH
  MULTICAST
}

input MessageInput {
  type: String!
  text: String
  # Flex message
  altText: String
  flexContent: JSON
}

# Payload types
type CreateOrderPayload {
  order: Order!
  paymentUrl: String
  linePayPaymentUrl: String
}

type CancelOrderPayload {
  order: Order!
  refundAmount: Float
}

type SendMessagePayload {
  success: Boolean!
  messageIds: [String!]!
}

type RedeemCouponPayload {
  coupon: Coupon!
  discountAmount: Float!
  newTotal: Float!
}
```

### 2.4 Subscription Types

```graphql
# schema/subscription.graphql

type Subscription {
  # Order subscriptions
  orderStatusChanged(orderId: ID!): OrderStatusChangedEvent!
  newOrder(userId: ID): NewOrderEvent!
  
  # Chat subscriptions
  newMessage(lineUserId: String!): ChatMessage!
  
  # Analytics subscriptions
  realtimeStats: RealtimeStats!
  
  # Stock subscriptions
  stockAlert(productId: ID!): StockAlertEvent!
}

type OrderStatusChangedEvent {
  order: Order!
  previousStatus: OrderStatus!
  newStatus: OrderStatus!
  changedAt: DateTime!
  changedBy: String
}

type NewOrderEvent {
  order: Order!
  user: LineUser!
}

type ChatMessage {
  id: ID!
  lineUserId: String!
  type: String!
  text: String
  timestamp: DateTime!
}

type RealtimeStats {
  activeUsers: Int!
  ordersToday: Int!
  revenueToday: Float!
  timestamp: DateTime!
}

type StockAlertEvent {
  product: Product!
  stockQuantity: Int!
  alertType: StockAlertType!
}

enum StockAlertType {
  LOW_STOCK
  OUT_OF_STOCK
  BACK_IN_STOCK
}
```

---

## 3. Apollo Server 4 Setup

### 3.1 Installation และ Configuration

```bash
npm install @apollo/server graphql graphql-tag
npm install @graphql-tools/merge @graphql-tools/load
npm install dataloader
npm install graphql-ws ws
npm install graphql-depth-limit graphql-query-complexity
npm install @types/graphql
```

```typescript
// src/graphql/server.ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { loadFilesSync } from '@graphql-tools/load-files';
import { mergeTypeDefs, mergeResolvers } from '@graphql-tools/merge';
import express from 'express';
import http from 'http';
import path from 'path';
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-query-complexity';

import { resolvers as userResolvers } from './resolvers/user.resolver';
import { resolvers as orderResolvers } from './resolvers/order.resolver';
import { resolvers as productResolvers } from './resolvers/product.resolver';
import { resolvers as subscriptionResolvers } from './resolvers/subscription.resolver';
import { createDataLoaders } from './dataloaders';
import { authenticate } from '../middleware/auth.middleware';
import { GraphQLContext } from './context';

export async function createGraphQLServer() {
  const app = express();
  const httpServer = http.createServer(app);

  // Load and merge type definitions
  const typesArray = loadFilesSync(
    path.join(__dirname, './schema/**/*.graphql')
  );
  const typeDefs = mergeTypeDefs(typesArray);

  // Merge resolvers
  const resolvers = mergeResolvers([
    userResolvers,
    orderResolvers,
    productResolvers,
    subscriptionResolvers,
  ]);

  // Create executable schema
  const schema = makeExecutableSchema({ typeDefs, resolvers });

  // WebSocket Server for subscriptions
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql',
  });

  const serverCleanup = useServer(
    {
      schema,
      context: async (ctx) => {
        const token = ctx.connectionParams?.Authorization as string;
        const user = token ? await authenticate(token) : null;
        
        return {
          user,
          dataLoaders: createDataLoaders(),
          pubsub: global.pubsub,
        };
      },
    },
    wsServer
  );

  // Apollo Server
  const server = new ApolloServer<GraphQLContext>({
    schema,
    validationRules: [
      depthLimit(10), // ป้องกัน deeply nested queries
      createComplexityLimitRule(1000, {
        onCost: (cost) => {
          console.log(`Query cost: ${cost}`);
        },
      }),
    ],
    plugins: [
      ApolloServerPluginDrainHttpServer({ httpServer }),
      {
        async serverWillStart() {
          return {
            async drainServer() {
              await serverCleanup.dispose();
            },
          };
        },
      },
      // Custom plugin for logging
      {
        async requestDidStart(requestContext) {
          const startTime = Date.now();
          
          return {
            async willSendResponse(responseContext) {
              const duration = Date.now() - startTime;
              console.log({
                operationName: requestContext.request.operationName,
                duration,
                errors: responseContext.response.body.kind === 'single'
                  ? responseContext.response.body.singleResult.errors?.length
                  : 0,
              });
            },
          };
        },
      },
    ],
    introspection: process.env.NODE_ENV !== 'production',
    formatError: (error) => {
      // ซ่อน internal errors ใน production
      if (process.env.NODE_ENV === 'production') {
        if (error.extensions?.code === 'INTERNAL_SERVER_ERROR') {
          return { message: 'Internal server error', extensions: { code: 'INTERNAL_SERVER_ERROR' } };
        }
      }
      return error;
    },
  });

  await server.start();

  app.use(
    '/graphql',
    express.json(),
    expressMiddleware(server, {
      context: async ({ req }) => {
        const token = req.headers.authorization?.replace('Bearer ', '');
        const user = token ? await authenticate(token) : null;
        
        return {
          user,
          dataLoaders: createDataLoaders(),
          pubsub: global.pubsub,
          req,
        };
      },
    })
  );

  return { app, httpServer, server };
}

// Context type
export interface GraphQLContext {
  user: any | null;
  dataLoaders: ReturnType<typeof createDataLoaders>;
  pubsub: any;
  req?: express.Request;
}
```

---

## 4. Resolvers สำหรับ LINE API

### 4.1 User Resolver

```typescript
// graphql/resolvers/user.resolver.ts
import { GraphQLContext } from '../context';
import { LineUserService } from '../../services/line-user.service';
import { AuthenticationError, ForbiddenError } from '../errors';

const userService = new LineUserService();

export const resolvers = {
  Query: {
    me: async (_: any, __: any, context: GraphQLContext) => {
      if (!context.user) {
        throw new AuthenticationError('กรุณาเข้าสู่ระบบก่อน');
      }
      return userService.getUserByLineId(context.user.lineUserId);
    },

    user: async (_: any, { id }: { id: string }, context: GraphQLContext) => {
      // ตรวจสอบ permissions
      if (!context.user?.isAdmin) {
        throw new ForbiddenError('ไม่มีสิทธิ์เข้าถึงข้อมูลผู้ใช้อื่น');
      }
      return userService.getUserById(id);
    },

    users: async (
      _: any,
      args: { tier?: string; search?: string; limit?: number; offset?: number },
      context: GraphQLContext
    ) => {
      if (!context.user?.isAdmin) {
        throw new ForbiddenError('ไม่มีสิทธิ์');
      }

      const [nodes, totalCount] = await Promise.all([
        userService.getUsers(args),
        userService.countUsers(args),
      ]);

      return {
        nodes,
        totalCount,
        pageInfo: {
          hasNextPage: (args.offset || 0) + nodes.length < totalCount,
          hasPreviousPage: (args.offset || 0) > 0,
        },
      };
    },
  },

  Mutation: {
    updateProfile: async (
      _: any,
      { input }: { input: any },
      context: GraphQLContext
    ) => {
      if (!context.user) {
        throw new AuthenticationError('กรุณาเข้าสู่ระบบก่อน');
      }
      return userService.updateProfile(context.user.id, input);
    },

    addAddress: async (
      _: any,
      { input }: { input: any },
      context: GraphQLContext
    ) => {
      if (!context.user) throw new AuthenticationError();
      return userService.addAddress(context.user.id, input);
    },
  },

  LineUser: {
    // Field resolvers ใช้ DataLoader ป้องกัน N+1
    orders: async (
      parent: any,
      args: { status?: string; limit?: number; offset?: number },
      context: GraphQLContext
    ) => {
      return context.dataLoaders.userOrdersLoader.load({
        userId: parent.id,
        ...args,
      });
    },

    activeCoupons: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.userCouponsLoader.load(parent.id);
    },

    cart: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.userCartLoader.load(parent.id);
    },

    loyaltyPoints: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.userLoyaltyLoader.load(parent.id);
    },
  },
};
```

### 4.2 Order Resolver

```typescript
// graphql/resolvers/order.resolver.ts
import { GraphQLContext } from '../context';
import { OrderService } from '../../services/order.service';
import { NotFoundError, ForbiddenError, ValidationError } from '../errors';
import { PubSubEvents } from '../subscriptions/pubsub';

const orderService = new OrderService();

export const resolvers = {
  Query: {
    order: async (_: any, { id }: { id: string }, context: GraphQLContext) => {
      const order = await orderService.getOrderById(id);
      
      if (!order) throw new NotFoundError('ไม่พบคำสั่งซื้อ');
      
      // ตรวจสอบว่าเป็นเจ้าของ order
      if (!context.user?.isAdmin && order.userId !== context.user?.id) {
        throw new ForbiddenError('ไม่มีสิทธิ์ดูคำสั่งซื้อนี้');
      }
      
      return order;
    },

    orders: async (_: any, args: any, context: GraphQLContext) => {
      // Admin ดูได้ทุก order, user ดูได้แค่ของตัวเอง
      const userId = context.user?.isAdmin ? args.userId : context.user?.id;
      
      const [nodes, totalCount] = await Promise.all([
        orderService.getOrders({ ...args, userId }),
        orderService.countOrders({ ...args, userId }),
      ]);

      return {
        nodes,
        totalCount,
        pageInfo: {
          hasNextPage: (args.offset || 0) + nodes.length < totalCount,
          hasPreviousPage: (args.offset || 0) > 0,
        },
      };
    },
  },

  Mutation: {
    createOrder: async (
      _: any,
      { input }: { input: any },
      context: GraphQLContext
    ) => {
      if (!context.user) throw new Error('กรุณาเข้าสู่ระบบ');

      // Validate stock
      const stockCheck = await orderService.checkStock(input.items);
      if (!stockCheck.available) {
        throw new ValidationError(
          `สินค้า ${stockCheck.unavailableItems.join(', ')} หมดสต๊อก`
        );
      }

      const order = await orderService.createOrder({
        ...input,
        userId: context.user.id,
        lineUserId: context.user.lineUserId,
      });

      // Publish subscription event
      await context.pubsub.publish(PubSubEvents.NEW_ORDER, {
        newOrder: { order, user: context.user },
      });

      // Generate payment URL
      let paymentUrl = null;
      let linePayPaymentUrl = null;

      if (input.paymentMethod === 'LINE_PAY') {
        linePayPaymentUrl = await orderService.initiateLinePayPayment(order.id);
      } else if (input.paymentMethod === 'CREDIT_CARD') {
        paymentUrl = await orderService.initiateCardPayment(order.id);
      }

      return { order, paymentUrl, linePayPaymentUrl };
    },

    cancelOrder: async (
      _: any,
      { id, reason }: { id: string; reason?: string },
      context: GraphQLContext
    ) => {
      const order = await orderService.getOrderById(id);
      if (!order) throw new NotFoundError('ไม่พบคำสั่งซื้อ');

      if (order.userId !== context.user?.id && !context.user?.isAdmin) {
        throw new ForbiddenError('ไม่มีสิทธิ์ยกเลิกคำสั่งซื้อนี้');
      }

      const cancelledOrder = await orderService.cancelOrder(id, reason);
      
      // Publish event
      await context.pubsub.publish(PubSubEvents.ORDER_STATUS_CHANGED, {
        orderStatusChanged: {
          order: cancelledOrder,
          previousStatus: order.status,
          newStatus: 'CANCELLED',
          changedAt: new Date(),
        },
      });

      return {
        order: cancelledOrder,
        refundAmount: order.totalAmount,
      };
    },
  },

  Order: {
    user: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.userLoader.load(parent.userId);
    },

    items: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.orderItemsLoader.load(parent.id);
    },

    payment: async (parent: any, _: any, context: GraphQLContext) => {
      return context.dataLoaders.orderPaymentLoader.load(parent.id);
    },

    shippingAddress: async (parent: any, _: any, context: GraphQLContext) => {
      if (!parent.shippingAddressId) return null;
      return context.dataLoaders.addressLoader.load(parent.shippingAddressId);
    },

    canCancel: (parent: any) => {
      return ['PENDING', 'CONFIRMED'].includes(parent.status);
    },

    canReorder: (parent: any) => {
      return ['DELIVERED', 'CANCELLED'].includes(parent.status);
    },
  },
};
```

---

## 5. DataLoader ป้องกัน N+1

### 5.1 DataLoader Implementation

```typescript
// graphql/dataloaders/index.ts
import DataLoader from 'dataloader';
import { Pool } from 'pg';

export function createDataLoaders() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });

  return {
    // User loaders
    userLoader: createUserLoader(pool),
    userOrdersLoader: createUserOrdersLoader(pool),
    userCouponsLoader: createUserCouponsLoader(pool),
    userCartLoader: createUserCartLoader(pool),
    userLoyaltyLoader: createUserLoyaltyLoader(pool),

    // Order loaders
    orderItemsLoader: createOrderItemsLoader(pool),
    orderPaymentLoader: createOrderPaymentLoader(pool),

    // Product loaders
    productLoader: createProductLoader(pool),
    productCategoryLoader: createProductCategoryLoader(pool),
    productImagesLoader: createProductImagesLoader(pool),

    // Address loaders
    addressLoader: createAddressLoader(pool),
  };
}

function createUserLoader(pool: Pool) {
  return new DataLoader<string, any>(async (userIds) => {
    const result = await pool.query(
      `SELECT * FROM users WHERE id = ANY($1)`,
      [userIds]
    );

    // สร้าง map สำหรับ lookup O(1)
    const userMap = new Map(result.rows.map(u => [u.id, u]));
    return userIds.map(id => userMap.get(id) || null);
  });
}

function createUserOrdersLoader(pool: Pool) {
  // Key เป็น object ต้องใช้ cacheKeyFn
  return new DataLoader<{ userId: string; status?: string; limit?: number }, any[]>(
    async (keys) => {
      const userIds = [...new Set(keys.map(k => k.userId))];
      
      const result = await pool.query(
        `SELECT * FROM orders 
         WHERE user_id = ANY($1) 
         ORDER BY created_at DESC`,
        [userIds]
      );

      return keys.map(key => {
        let orders = result.rows.filter(o => o.user_id === key.userId);
        
        if (key.status) {
          orders = orders.filter(o => o.status === key.status);
        }
        
        if (key.limit) {
          orders = orders.slice(0, key.limit);
        }
        
        return orders;
      });
    },
    {
      cacheKeyFn: (key) => `${key.userId}:${key.status || 'all'}:${key.limit || 10}`,
    }
  );
}

function createOrderItemsLoader(pool: Pool) {
  return new DataLoader<string, any[]>(async (orderIds) => {
    const result = await pool.query(
      `SELECT 
        oi.*,
        p.name as product_name,
        p.price as product_price,
        p.primary_image_url
       FROM order_items oi
       JOIN products p ON oi.product_id = p.id
       WHERE oi.order_id = ANY($1)`,
      [orderIds]
    );

    const itemsByOrder = new Map<string, any[]>();
    
    result.rows.forEach(item => {
      const existing = itemsByOrder.get(item.order_id) || [];
      existing.push(item);
      itemsByOrder.set(item.order_id, existing);
    });

    return orderIds.map(id => itemsByOrder.get(id) || []);
  });
}

function createProductLoader(pool: Pool) {
  return new DataLoader<string, any>(async (productIds) => {
    const result = await pool.query(
      `SELECT * FROM products WHERE id = ANY($1)`,
      [productIds]
    );

    const productMap = new Map(result.rows.map(p => [p.id, p]));
    return productIds.map(id => productMap.get(id) || null);
  });
}

function createProductImagesLoader(pool: Pool) {
  return new DataLoader<string, any[]>(async (productIds) => {
    const result = await pool.query(
      `SELECT * FROM product_images 
       WHERE product_id = ANY($1) 
       ORDER BY sort_order`,
      [productIds]
    );

    const imagesByProduct = new Map<string, any[]>();
    
    result.rows.forEach(image => {
      const existing = imagesByProduct.get(image.product_id) || [];
      existing.push(image);
      imagesByProduct.set(image.product_id, existing);
    });

    return productIds.map(id => imagesByProduct.get(id) || []);
  });
}

function createUserCouponsLoader(pool: Pool) {
  return new DataLoader<string, any[]>(async (userIds) => {
    const result = await pool.query(
      `SELECT uc.*, c.* 
       FROM user_coupons uc
       JOIN coupons c ON uc.coupon_id = c.id
       WHERE uc.user_id = ANY($1)
       AND c.expires_at > NOW()
       AND uc.used_at IS NULL`,
      [userIds]
    );

    const couponsByUser = new Map<string, any[]>();
    
    result.rows.forEach(coupon => {
      const existing = couponsByUser.get(coupon.user_id) || [];
      existing.push(coupon);
      couponsByUser.set(coupon.user_id, existing);
    });

    return userIds.map(id => couponsByUser.get(id) || []);
  });
}

function createUserCartLoader(pool: Pool) {
  return new DataLoader<string, any>(async (userIds) => {
    const result = await pool.query(
      `SELECT c.*, 
        json_agg(
          json_build_object(
            'id', ci.id,
            'productId', ci.product_id,
            'quantity', ci.quantity
          )
        ) as items
       FROM carts c
       LEFT JOIN cart_items ci ON c.id = ci.cart_id
       WHERE c.user_id = ANY($1)
       GROUP BY c.id`,
      [userIds]
    );

    const cartMap = new Map(result.rows.map(c => [c.user_id, c]));
    return userIds.map(id => cartMap.get(id) || null);
  });
}

function createUserLoyaltyLoader(pool: Pool) {
  return new DataLoader<string, number>(async (userIds) => {
    const result = await pool.query(
      `SELECT user_id, SUM(points) as total_points
       FROM loyalty_transactions
       WHERE user_id = ANY($1)
       GROUP BY user_id`,
      [userIds]
    );

    const pointsMap = new Map(result.rows.map(r => [r.user_id, parseInt(r.total_points)]));
    return userIds.map(id => pointsMap.get(id) || 0);
  });
}

function createOrderPaymentLoader(pool: Pool) {
  return new DataLoader<string, any>(async (orderIds) => {
    const result = await pool.query(
      `SELECT * FROM payments WHERE order_id = ANY($1)`,
      [orderIds]
    );

    const paymentMap = new Map(result.rows.map(p => [p.order_id, p]));
    return orderIds.map(id => paymentMap.get(id) || null);
  });
}

function createAddressLoader(pool: Pool) {
  return new DataLoader<string, any>(async (addressIds) => {
    const result = await pool.query(
      `SELECT * FROM addresses WHERE id = ANY($1)`,
      [addressIds]
    );

    const addressMap = new Map(result.rows.map(a => [a.id, a]));
    return addressIds.map(id => addressMap.get(id) || null);
  });
}

function createProductCategoryLoader(pool: Pool) {
  return new DataLoader<string, any>(async (categoryIds) => {
    const result = await pool.query(
      `SELECT * FROM categories WHERE id = ANY($1)`,
      [categoryIds]
    );

    const categoryMap = new Map(result.rows.map(c => [c.id, c]));
    return categoryIds.map(id => categoryMap.get(id) || null);
  });
}
```

---

## 6. Subscriptions สำหรับ Real-time Updates

### 6.1 PubSub Setup

```typescript
// graphql/subscriptions/pubsub.ts
import { PubSub } from 'graphql-subscriptions';
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

export enum PubSubEvents {
  ORDER_STATUS_CHANGED = 'ORDER_STATUS_CHANGED',
  NEW_ORDER = 'NEW_ORDER',
  NEW_MESSAGE = 'NEW_MESSAGE',
  REALTIME_STATS = 'REALTIME_STATS',
  STOCK_ALERT = 'STOCK_ALERT',
}

// ใช้ Redis PubSub สำหรับ multi-instance setup
function createRedisPubSub(): RedisPubSub {
  const options = {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
    password: process.env.REDIS_PASSWORD,
    retryStrategy: (times: number) => Math.min(times * 50, 2000),
  };

  return new RedisPubSub({
    publisher: new Redis(options),
    subscriber: new Redis(options),
  });
}

export const pubsub = process.env.NODE_ENV === 'development'
  ? new PubSub()
  : createRedisPubSub();

// ประกาศ global
declare global {
  var pubsub: typeof pubsub;
}
global.pubsub = pubsub;
```

### 6.2 Subscription Resolvers

```typescript
// graphql/resolvers/subscription.resolver.ts
import { withFilter } from 'graphql-subscriptions';
import { PubSubEvents } from '../subscriptions/pubsub';
import { GraphQLContext } from '../context';

export const resolvers = {
  Subscription: {
    orderStatusChanged: {
      subscribe: withFilter(
        (_: any, __: any, context: GraphQLContext) =>
          context.pubsub.asyncIterator(PubSubEvents.ORDER_STATUS_CHANGED),
        (payload: any, variables: any, context: GraphQLContext) => {
          // Filter: ส่งเฉพาะ order ที่ user เป็นเจ้าของ
          const order = payload.orderStatusChanged.order;
          return (
            order.id === variables.orderId ||
            (context.user?.isAdmin === true)
          );
        }
      ),
    },

    newOrder: {
      subscribe: withFilter(
        (_: any, __: any, context: GraphQLContext) =>
          context.pubsub.asyncIterator(PubSubEvents.NEW_ORDER),
        (payload: any, variables: any, context: GraphQLContext) => {
          // Admin เห็นทุก order, user เห็นเฉพาะของตัวเอง
          if (context.user?.isAdmin) return true;
          
          if (variables.userId) {
            return payload.newOrder.order.userId === variables.userId;
          }
          
          return payload.newOrder.order.userId === context.user?.id;
        }
      ),
    },

    newMessage: {
      subscribe: withFilter(
        (_: any, __: any, context: GraphQLContext) =>
          context.pubsub.asyncIterator(PubSubEvents.NEW_MESSAGE),
        (payload: any, variables: any) =>
          payload.newMessage.lineUserId === variables.lineUserId
      ),
    },

    realtimeStats: {
      subscribe: (_: any, __: any, context: GraphQLContext) => {
        if (!context.user?.isAdmin) {
          throw new Error('ต้องเป็น Admin เท่านั้น');
        }
        return context.pubsub.asyncIterator(PubSubEvents.REALTIME_STATS);
      },
    },

    stockAlert: {
      subscribe: withFilter(
        (_: any, __: any, context: GraphQLContext) =>
          context.pubsub.asyncIterator(PubSubEvents.STOCK_ALERT),
        (payload: any, variables: any) =>
          payload.stockAlert.product.id === variables.productId
      ),
    },
  },
};
```

---

## 7. GraphQL กับ LIFF Frontend

### 7.1 Apollo Client ใน LIFF

```typescript
// liff/apollo-client.ts
import { ApolloClient, InMemoryCache, split, HttpLink, from } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { createClient } from 'graphql-ws';
import { getMainDefinition } from '@apollo/client/utilities';
import { onError } from '@apollo/client/link/error';
import { setContext } from '@apollo/client/link/context';
import liff from '@line/liff';

// Auth link - เพิ่ม LINE access token
const authLink = setContext(async (_, { headers }) => {
  const accessToken = liff.getAccessToken();
  
  return {
    headers: {
      ...headers,
      authorization: accessToken ? `Bearer ${accessToken}` : '',
    },
  };
});

// HTTP Link สำหรับ queries และ mutations
const httpLink = new HttpLink({
  uri: process.env.REACT_APP_GRAPHQL_URL || 'https://api.yourapp.com/graphql',
});

// WebSocket Link สำหรับ subscriptions
const wsLink = new GraphQLWsLink(
  createClient({
    url: process.env.REACT_APP_GRAPHQL_WS_URL || 'wss://api.yourapp.com/graphql',
    connectionParams: async () => {
      const accessToken = liff.getAccessToken();
      return {
        Authorization: accessToken ? `Bearer ${accessToken}` : '',
      };
    },
    retryAttempts: 5,
  })
);

// Error handling link
const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    graphQLErrors.forEach(({ message, locations, path, extensions }) => {
      console.error(`GraphQL Error: ${message}`, { locations, path });
      
      if (extensions?.code === 'UNAUTHENTICATED') {
        // Redirect to LINE login
        liff.login();
      }
    });
  }

  if (networkError) {
    console.error('Network Error:', networkError);
  }
});

// Split: ใช้ WebSocket สำหรับ subscriptions, HTTP สำหรับ queries/mutations
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  from([authLink, errorLink, httpLink])
);

export const apolloClient = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          orders: {
            // Cache pagination
            keyArgs: ['userId', 'status'],
            merge(existing, incoming) {
              return {
                ...incoming,
                nodes: [...(existing?.nodes || []), ...incoming.nodes],
              };
            },
          },
        },
      },
      Order: {
        fields: {
          canCancel: {
            read(_, { readField }) {
              const status = readField('status');
              return ['PENDING', 'CONFIRMED'].includes(status as string);
            },
          },
        },
      },
    },
  }),
  defaultOptions: {
    watchQuery: {
      fetchPolicy: 'cache-and-network',
    },
  },
});
```

### 7.2 LIFF Components ด้วย Apollo

```tsx
// liff/components/OrderHistory.tsx
import React, { useState } from 'react';
import { useQuery, useSubscription, gql } from '@apollo/client';

const GET_MY_ORDERS = gql`
  query GetMyOrders($status: OrderStatus, $limit: Int, $offset: Int) {
    orders(status: $status, limit: $limit, offset: $offset) {
      nodes {
        id
        orderNumber
        status
        totalAmount
        createdAt
        items {
          product {
            name
            primaryImage {
              url
            }
          }
          quantity
          totalPrice
        }
        canCancel
      }
      totalCount
      pageInfo {
        hasNextPage
      }
    }
  }
`;

const ORDER_STATUS_SUBSCRIPTION = gql`
  subscription OnOrderStatusChanged($orderId: ID!) {
    orderStatusChanged(orderId: $orderId) {
      order {
        id
        status
      }
      previousStatus
      newStatus
      changedAt
    }
  }
`;

const CANCEL_ORDER = gql`
  mutation CancelOrder($id: ID!, $reason: String) {
    cancelOrder(id: $id, reason: $reason) {
      order {
        id
        status
      }
      refundAmount
    }
  }
`;

export const OrderHistory: React.FC = () => {
  const [selectedStatus, setSelectedStatus] = useState<string | undefined>();
  const [page, setPage] = useState(0);
  const PAGE_SIZE = 10;

  const { data, loading, error, fetchMore } = useQuery(GET_MY_ORDERS, {
    variables: {
      status: selectedStatus,
      limit: PAGE_SIZE,
      offset: page * PAGE_SIZE,
    },
  });

  if (loading) return <div className="loading">กำลังโหลด...</div>;
  if (error) return <div className="error">เกิดข้อผิดพลาด: {error.message}</div>;

  const orders = data?.orders.nodes || [];
  const hasNextPage = data?.orders.pageInfo.hasNextPage;

  return (
    <div className="order-history">
      <h2>ประวัติการสั่งซื้อ</h2>

      {/* Status Filter */}
      <div className="status-filter">
        {['', 'PENDING', 'PAID', 'SHIPPED', 'DELIVERED'].map(status => (
          <button
            key={status}
            className={selectedStatus === status || (!selectedStatus && !status) ? 'active' : ''}
            onClick={() => {
              setSelectedStatus(status || undefined);
              setPage(0);
            }}
          >
            {getStatusLabel(status)}
          </button>
        ))}
      </div>

      {/* Order List */}
      {orders.map((order: any) => (
        <OrderCard key={order.id} order={order} />
      ))}

      {/* Load More */}
      {hasNextPage && (
        <button
          onClick={() => {
            fetchMore({
              variables: { offset: orders.length },
              updateQuery: (prev, { fetchMoreResult }) => {
                if (!fetchMoreResult) return prev;
                return {
                  orders: {
                    ...fetchMoreResult.orders,
                    nodes: [
                      ...prev.orders.nodes,
                      ...fetchMoreResult.orders.nodes,
                    ],
                  },
                };
              },
            });
          }}
        >
          โหลดเพิ่มเติม
        </button>
      )}
    </div>
  );
};

// Order Card ที่มี real-time status update
const OrderCard: React.FC<{ order: any }> = ({ order }) => {
  const { data: subscriptionData } = useSubscription(ORDER_STATUS_SUBSCRIPTION, {
    variables: { orderId: order.id },
    skip: ['DELIVERED', 'CANCELLED', 'REFUNDED'].includes(order.status),
  });

  const currentStatus = subscriptionData?.orderStatusChanged.newStatus || order.status;

  return (
    <div className="order-card">
      <div className="order-header">
        <span className="order-number">#{order.orderNumber}</span>
        <span className={`status status-${currentStatus.toLowerCase()}`}>
          {getStatusLabel(currentStatus)}
        </span>
      </div>

      <div className="order-items">
        {order.items.slice(0, 2).map((item: any) => (
          <div key={item.product.name} className="order-item">
            <img
              src={item.product.primaryImage?.url}
              alt={item.product.name}
              className="product-image"
            />
            <div>
              <p>{item.product.name}</p>
              <p>x{item.quantity}</p>
            </div>
            <p>฿{item.totalPrice.toLocaleString()}</p>
          </div>
        ))}
        {order.items.length > 2 && (
          <p>+{order.items.length - 2} รายการอื่น</p>
        )}
      </div>

      <div className="order-footer">
        <span>รวม: ฿{order.totalAmount.toLocaleString()}</span>
        <span>{new Date(order.createdAt).toLocaleDateString('th-TH')}</span>
      </div>

      {order.canCancel && (
        <button
          className="cancel-btn"
          onClick={() => handleCancelOrder(order.id)}
        >
          ยกเลิกคำสั่งซื้อ
        </button>
      )}
    </div>
  );
};

function getStatusLabel(status: string): string {
  const labels: Record<string, string> = {
    '': 'ทั้งหมด',
    'PENDING': 'รอดำเนินการ',
    'CONFIRMED': 'ยืนยันแล้ว',
    'PAID': 'ชำระเงินแล้ว',
    'PREPARING': 'กำลังเตรียม',
    'SHIPPED': 'กำลังจัดส่ง',
    'DELIVERED': 'ส่งแล้ว',
    'CANCELLED': 'ยกเลิก',
    'REFUNDED': 'คืนเงินแล้ว',
  };
  return labels[status] || status;
}

function handleCancelOrder(orderId: string) {
  // Implement cancel logic
}
```

---

## 8. Apollo Federation สำหรับ Microservices

### 8.1 Federation Setup

```typescript
// gateway/federation-gateway.ts
import { ApolloGateway, IntrospectAndCompose, RemoteGraphQLDataSource } from '@apollo/gateway';
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';

class AuthenticatedDataSource extends RemoteGraphQLDataSource {
  willSendRequest({ request, context }: any) {
    // ส่ง auth headers ไปยัง subgraph
    request.http.headers.set('x-user-id', context.userId);
    request.http.headers.set('x-user-role', context.userRole);
    request.http.headers.set('authorization', context.token || '');
  }
}

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://user-service:4001/graphql' },
      { name: 'orders', url: 'http://order-service:4002/graphql' },
      { name: 'products', url: 'http://product-service:4003/graphql' },
      { name: 'analytics', url: 'http://analytics-service:4004/graphql' },
      { name: 'line', url: 'http://line-service:4005/graphql' },
    ],
  }),
  buildService({ url }) {
    return new AuthenticatedDataSource({ url });
  },
});

const server = new ApolloServer({
  gateway,
});

const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => {
    return {
      token: req.headers.authorization,
      userId: req.headers['x-user-id'],
      userRole: req.headers['x-user-role'],
    };
  },
  listen: { port: 4000 },
});

console.log(`Gateway running at ${url}`);
```

### 8.2 Subgraph Schema (User Service)

```typescript
// user-service/schema.ts
import { buildSubgraphSchema } from '@apollo/subgraph';
import gql from 'graphql-tag';

const typeDefs = gql`
  extend schema @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])

  type User @key(fields: "id") {
    id: ID!
    lineUserId: String!
    displayName: String!
    email: String
    loyaltyPoints: Int!
  }

  type Query {
    me: User
    user(id: ID!): User
  }
`;

const resolvers = {
  User: {
    __resolveReference: async (user: { id: string }, context: any) => {
      return await getUserById(user.id);
    },
  },
  Query: {
    me: async (_: any, __: any, context: any) => {
      return getUserById(context.userId);
    },
    user: async (_: any, { id }: any, context: any) => {
      if (context.userRole !== 'admin') throw new Error('Forbidden');
      return getUserById(id);
    },
  },
};

export const schema = buildSubgraphSchema({ typeDefs, resolvers });
```

---

## 9. GraphQL Security

### 9.1 Depth Limiting และ Complexity

```typescript
// graphql/security/rules.ts
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-query-complexity';
import { fieldExtensionsEstimator, simpleEstimator } from 'graphql-query-complexity';

export const securityRules = [
  // ป้องกัน deeply nested queries
  depthLimit(10, { ignore: ['__schema', '__type'] }),
  
  // ป้องกัน expensive queries
  createComplexityLimitRule(1000, {
    estimators: [
      fieldExtensionsEstimator(),
      simpleEstimator({ defaultComplexity: 1 }),
    ],
    formatErrorMessage: (cost) =>
      `Query too complex: ${cost}. Maximum allowed: 1000`,
    onCost: (cost, args) => {
      if (cost > 500) {
        console.warn(`High complexity query: ${cost}`, {
          operation: args.operationName,
        });
      }
    },
  }),
];

// Rate limiting per user
import { RateLimiterRedis } from 'rate-limiter-flexible';
import Redis from 'ioredis';

const redisClient = new Redis(process.env.REDIS_URL);

export const queryRateLimiter = new RateLimiterRedis({
  storeClient: redisClient,
  keyPrefix: 'graphql_rate_limit',
  points: 100,   // 100 queries
  duration: 60,  // per 60 seconds
  blockDuration: 60,
});

// Middleware สำหรับ rate limiting
export async function rateLimitMiddleware(
  req: any,
  res: any,
  next: any
): Promise<void> {
  const userId = req.user?.id || req.ip;
  
  try {
    await queryRateLimiter.consume(userId);
    next();
  } catch (rateLimiterRes) {
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: Math.ceil((rateLimiterRes as any).msBeforeNext / 1000),
    });
  }
}
```

### 9.2 Persisted Queries

```typescript
// graphql/security/persisted-queries.ts
import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { sha256 } from 'crypto-hash';

// Client-side: ส่งเฉพาะ hash ของ query
export const persistedQueriesLink = createPersistedQueryLink({
  sha256,
  useGETForHashedQueries: true,
});

// Server-side: whitelist ของ queries ที่อนุญาต
const ALLOWED_QUERIES = new Map([
  ['abc123', 'query GetMyOrders { orders { nodes { id status } } }'],
  ['def456', 'query GetProduct($id: ID!) { product(id: $id) { id name price } }'],
]);

export function persistedQueryPlugin() {
  return {
    async requestDidStart() {
      return {
        async parsingDidStart({ queryString, operationName }: any) {
          if (process.env.NODE_ENV === 'production') {
            // ใน production ยอมรับเฉพาะ persisted queries
            // ตรวจสอบว่า query อยู่ใน whitelist
          }
        },
      };
    },
  };
}
```

---

## 10. Complete GraphQL API Implementation

### 10.1 Product Resolver

```typescript
// graphql/resolvers/product.resolver.ts

export const resolvers = {
  Query: {
    product: async (_: any, { id }: { id: string }, context: GraphQLContext) => {
      const product = await productService.getById(id);
      if (!product) throw new NotFoundError('ไม่พบสินค้า');
      return product;
    },

    products: async (_: any, args: any) => {
      const [nodes, totalCount] = await Promise.all([
        productService.search(args),
        productService.count(args),
      ]);

      return {
        nodes,
        totalCount,
        pageInfo: {
          hasNextPage: (args.offset || 0) + nodes.length < totalCount,
          hasPreviousPage: (args.offset || 0) > 0,
        },
      };
    },
  },

  Product: {
    category: (parent: any, _: any, context: GraphQLContext) =>
      context.dataLoaders.productCategoryLoader.load(parent.categoryId),

    images: (parent: any, _: any, context: GraphQLContext) =>
      context.dataLoaders.productImagesLoader.load(parent.id),

    primaryImage: async (parent: any, _: any, context: GraphQLContext) => {
      const images = await context.dataLoaders.productImagesLoader.load(parent.id);
      return images[0] || null;
    },

    isAvailable: (parent: any) => parent.stockQuantity > 0,

    reviewCount: async (parent: any, _: any, context: GraphQLContext) => {
      return reviewService.countByProduct(parent.id);
    },

    averageRating: async (parent: any, _: any, context: GraphQLContext) => {
      return reviewService.averageRatingByProduct(parent.id);
    },
  },
};
```

### 10.2 Analytics Resolver

```typescript
// graphql/resolvers/analytics.resolver.ts

export const resolvers = {
  Query: {
    analytics: async (
      _: any,
      { dateFrom, dateTo }: { dateFrom: Date; dateTo: Date },
      context: GraphQLContext
    ) => {
      if (!context.user?.isAdmin) {
        throw new ForbiddenError('ต้องเป็น Admin');
      }

      const [
        totalOrders,
        totalRevenue,
        newUsers,
        activeUsers,
        topProducts,
        conversionRate,
      ] = await Promise.all([
        analyticsService.countOrders(dateFrom, dateTo),
        analyticsService.sumRevenue(dateFrom, dateTo),
        analyticsService.countNewUsers(dateFrom, dateTo),
        analyticsService.countActiveUsers(dateFrom, dateTo),
        analyticsService.getTopProducts(dateFrom, dateTo, 10),
        analyticsService.calculateConversionRate(dateFrom, dateTo),
      ]);

      return {
        dateFrom,
        dateTo,
        totalOrders,
        totalRevenue,
        newUsers,
        activeUsers,
        topProducts,
        conversionRate,
      };
    },
  },
};
```

### 10.3 Error Handling

```typescript
// graphql/errors/index.ts
import { GraphQLError } from 'graphql';

export class AuthenticationError extends GraphQLError {
  constructor(message = 'กรุณาเข้าสู่ระบบก่อน') {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED',
        http: { status: 401 },
      },
    });
  }
}

export class ForbiddenError extends GraphQLError {
  constructor(message = 'ไม่มีสิทธิ์เข้าถึง') {
    super(message, {
      extensions: {
        code: 'FORBIDDEN',
        http: { status: 403 },
      },
    });
  }
}

export class NotFoundError extends GraphQLError {
  constructor(message = 'ไม่พบข้อมูล') {
    super(message, {
      extensions: {
        code: 'NOT_FOUND',
        http: { status: 404 },
      },
    });
  }
}

export class ValidationError extends GraphQLError {
  constructor(message: string) {
    super(message, {
      extensions: {
        code: 'BAD_USER_INPUT',
        http: { status: 400 },
      },
    });
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **GraphQL Schema Design** สำหรับ LINE data types
2. **Apollo Server 4** setup พร้อม WebSocket subscriptions
3. **Resolvers** สำหรับ User, Order, Product
4. **DataLoader** ป้องกัน N+1 problem
5. **Real-time Subscriptions** ด้วย Redis PubSub
6. **LIFF Integration** กับ Apollo Client
7. **Apollo Federation** สำหรับ microservices
8. **Security** ด้วย depth limiting, complexity, persisted queries

GraphQL เหมาะมากสำหรับ LIFF apps ที่ต้องการข้อมูลหลากหลายรูปแบบ และ real-time features ด้วย subscriptions
