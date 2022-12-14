<template>
  <div>
    <Navbar />
    <div v-if="posts && posts.length > 0">
      <div class="container mt-4">
        <table class="table">
          <thead>
            <tr>
              <th scope="col">Product</th>
              <th scope="col">Quantity</th>
              <th scope="col">Price</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="post in posts" :key="post.id" class="">
              <td>{{ post.name }}</td>
              <td>
                <button
                  @click="decrementQuantity(post)"
                  class="btn btn-default"
                >
                  -
                </button>
                {{ getQuantity(post) }}
                <button
                  @click="incrementQuantity(post)"
                  class="btn btn-default"
                >
                  +
                </button>
              </td>
              <td>USD {{ getPrice(post) }}</td>
            </tr>
          </tbody>
        </table>
        <div>
          <h4>Total price to be charged: USD {{ getTotalPrice() }}</h4>
          <stripe-checkout
            ref="checkoutRef"
            mode="payment"
            :pk="pk"
            :line-items="lineItems"
            :successUrl="successUrl"
            :cancelUrl="cancelUrl"
            @loading="(v) => (loading = v)"
          />
          <button @click="checkout">Checkout</button>
        </div>
      </div>
    </div>
    <div v-else-if="posts && posts.length == 0">
      <div class="col-md-12 jumbotron jumbotron-fluid mt-4">
        <div class="container">
          <h1 class="display-4">Loading...</h1>
          <p class="lead">Hold on as we get the cart items...</p>
        </div>
      </div>
    </div>
    <div v-else>
      <div class="col-md-12 jumbotron jumbotron-fluid mt-4">
        <div class="container">
          <h1 class="display-4">No items</h1>
          <p class="lead">Your cart is Empty,</p>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import Navbar from "../components/Navbar.vue";
export default {
  components: {
    Navbar,
  },
  async mounted() {
    await this.createProductsFromCart();
    await this.createPricesFromCart();
  },
  methods: {
    incrementQuantity(product) {
      if (process.client) {
        // setting the data...
        let newCart = JSON.parse(localStorage.getItem("cartItems")).map(
          (cartItem) => {
            if (cartItem.id == product.id) {
              // do the mapping
              let new_quantity = parseInt(cartItem.quantity) + 1;
              let price_per_quantity =
                parseInt(cartItem.price) / parseInt(cartItem.quantity);
              let new_price = price_per_quantity * new_quantity;
              return {
                ...cartItem,
                quantity: new_quantity,
                price: new_price,
              };
            } else {
              return {
                ...cartItem,
              };
            }
          }
        );
        localStorage.setItem("cartItems", JSON.stringify(newCart));
        window.location.reload();
      }
    },
    decrementQuantity(product) {
      if (process.client && product.quantity > 1) {
        // setting the data...
        let newCart = JSON.parse(localStorage.getItem("cartItems")).map(
          (cartItem) => {
            if (cartItem.id == product.id) {
              // do the mapping
              let new_quantity = parseInt(cartItem.quantity) - 1;
              let price_per_quantity =
                parseInt(cartItem.price) / parseInt(cartItem.quantity);
              let new_price = price_per_quantity * new_quantity;
              return {
                ...cartItem,
                quantity: new_quantity,
                price: new_price,
              };
            } else {
              return {
                ...cartItem,
              };
            }
          }
        );
        localStorage.setItem("cartItems", JSON.stringify(newCart));
        window.location.reload();
      }
    },
    getQuantity(product) {
      return JSON.parse(localStorage.getItem("cartItems")).find(
        (item) => item.id == product.id
      ).quantity;
    },
    getPrice(product) {
      return JSON.parse(localStorage.getItem("cartItems")).find(
        (item) => item.id == product.id
      ).price;
    },
    getTotalPrice() {
      if (process.client) {
        let postItems = JSON.parse(localStorage.getItem("cartItems"));
        let prices = postItems.map((item) => item.price);
        return prices.reduce((a, b) => parseInt(a) + parseInt(b));
      } else {
        return 0.0;
      }
    },
    async createProductsFromCart() {
      let cartItems = JSON.parse(localStorage.getItem("cartItems"));
      if (cartItems.length > 0) {
        // loop through the cart items.
        for (const item of cartItems) {
          let response = await this.$axios.post(
            "http://localhost:3000/api/create-product",
            {
              name: item.name,
            }
          );
          // add the product id to cartItems . ..
          cartItems = cartItems.map((cartItem) => {
            if (cartItem.id == item.id) {
              return {
                ...cartItem,
                productId: response.data.product.id,
              };
            } else {
              return cartItem;
            }
          });
        }
        localStorage.setItem("cartItems", JSON.stringify(cartItems));
      }
    },
    async createPricesFromCart() {
      let cartItems = JSON.parse(localStorage.getItem("cartItems"));
      if (cartItems.length > 0) {
        // loop through the cart items.
        for (const item of cartItems) {
          let response = await this.$axios.post(
            "http://localhost:3000/api/create-price",
            {
              amount: item.price,
              productId: item.productId,
            }
          );
          // add the product id to cartItems . ..
          cartItems = cartItems.map((cartItem) => {
            if (cartItem.id == item.id) {
              return {
                ...cartItem,
                priceId: response.data.price.id,
              };
            } else {
              return cartItem;
            }
          });
        }
        localStorage.setItem("cartItems", JSON.stringify(cartItems));
      }
    },
    async generatePaymentIntent() {
      const paymentIntent = await this.$axios.post(
        "http://localhost:3000/api/create-payment-intent",
        {
          items: JSON.parse(localStorage.getItem("cartItems")),
        }
      );
      this.elementsOptions.clientSecret = paymentIntent.client_secret;
    },
    checkout() {
      this.$refs.checkoutRef.redirectToCheckout();
    },
  },
  data() {
    this.pk = process.env.STRIPE_PK;
    return {
      posts: process.client
        ? localStorage.getItem("cartItems")
          ? JSON.parse(localStorage.getItem("cartItems"))
          : []
        : [],
      successUrl: "http://localhost:3000/success",
      cancelUrl: "http://localhost:3000/error",
      lineItems: process.client
        ? JSON.parse(localStorage.getItem("cartItems")).map((item) => {
            console.log("on line items ", item);
            return {
              price: item.priceId,
              quantity: item.quantity,
            };
          })
        : [],
    };
  },
};
</script>
