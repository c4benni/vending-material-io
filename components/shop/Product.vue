<template>
  <div
    class="card min-h-[360px] lg:min-h-[510px] mx-auto w-full relative lg:p-2 p-0 grid lg:grid-flow-col lg:grid-cols-[auto,1fr] before-divide before:border fill-before before:opacity-[0.05]"
  >
    <div
      v-if="loading"
      class="h-full w-full absolute grid place-items-center top-0"
    >
      <Loader />
    </div>

    <template v-else>
      <div
        class="w-[calc(100%-1rem)] h-[360px] lg:w-[490px] lg:h-[510px] rounded-md bg-white dark:bg-blue-gray-900 shadow-md justify-self-center mt-2 lg:mt-0 isolate overflow-hidden"
      >
        <Img :public-id="product.background" class="h-full object-contain" />
      </div>
      <div
        class="grid lg:p-2 p-4 lg:pl-4 pt-6 lg:pt-6 pb-10 lg:pb-0 content-start"
      >
        <p class="text-sm font-medium opacity-70 capitalize">
          {{ product.type }}
        </p>
        <p class="text-xl font-bold py-1 capitalize">
          {{ product.productName }}
        </p>
        <p
          v-if="product.caption"
          class="text-[0.9rem] mb-2 font-light opacity-80"
        >
          {{ product.caption }}
        </p>

        <v-rating class="mx-3 mt-2" readonly :value="4" color="amber" />

        <div
          class="before-divide before:border-b fill-before pb-4 relative mt-3 flex justify-between items-center"
        >
          <p class="text-xl font-bold">{{ productCost }}</p>

          <div
            class="rounded-full px-2 py-1 bg-opacity-75 dark:bg-opacity-75 text-[0.8rem]"
            :class="{
              'bg-green-800 text-white': productStock >= 10,
              'bg-yellow-500 text-black': productStock < 10 && productStock > 4,
              'bg-red-800 text-white': productStock < 5
            }"
          >
            {{ stockBadge }}
          </div>
        </div>

        <p class="text-sm font-medium opacity-70 mt-4">Choose quantity</p>

        <Quantity
          v-model="quantity"
          :disabled="isSeller || soldOut"
          :max="productStock"
          class="justify-self-start mb-12"
        />

        <v-btn
          class="min-w-[200px] h-[48px] mx-auto grid w-[min(70%,300px)]"
          :class="{
            'bg-blue-800 text-white dark:bg-blue-400 dark:text-black':
              !disablePurchase && !ownerDeleted
          }"
          :disabled="disablePurchase || ownerDeleted"
          @click="purchase"
        >
          {{ purchaseText }}
          <div v-if="purchasing" class="spinner-border"></div>
        </v-btn>

        <p
          v-if="!isSeller"
          class="text-sm font-light mt-2 text-center opacity-70"
        >
          {{ purchaseHint }}
        </p>

        <p v-else class="text-sm font-light mt-2 text-center opacity-70">
          You need to create a buyer's account first.
        </p>

        <v-btn
          v-if="showDeposit"
          class="min-w-[200px] h-[48px] mx-auto grid w-[min(70%,300px)] text-blue-800 dark:text-blue-500 fade-appear hover:underline font-medium mt-6 ring-1"
to="/deposit"

          >Deposit coins</v-btn
        >
      </div>
    </template>
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex'

import { formatAmount } from '~/utils/main'
import buy from '~/services/buy'

export default {
  name: 'ShopProduct',

  data: () => ({
    purchasing: false,
    loading: true,
    quantity: 1
  }),

  head() {
    const productName = this.product.productName
    return {
      title: this.loading
        ? 'Loading product'
        : productName
        ? `Shop ${productName}`
        : 'Failed to fetch'
    }
  },

  computed: {
    ...mapState(['products']),

    product() {
      return this.products[this.id] || {}
    },

    ownerDeleted() {
      return this.product.ownerDeleted
    },
    purchaseHint() {
      if (this.product.ownerDeleted) {
        return 'Product owner deleted. Please try another product.'
      }
      return this.soldOut
        ? 'Item is out of stock. Check back later.'
        : this.insufficientDeposit
        ? 'Deposit more coins and try again.'
        : `You spend ${this.getTotalCost} in total`
    },
    stockBadge() {
      const stock = this.productStock

      return this.soldOut
        ? 'Out of stock'
        : `${stock} item${stock > 1 ? 's' : ''} left`
    },
    productCost() {
      return formatAmount(this.product.cost)
    },
    productStock() {
      return parseInt(this.product.amountAvailable || 0)
    },
    soldOut() {
      return !this.productStock
    },
    getTotalCost() {
      const cents = this.product.cost * this.quantity

      const toFixed = (val) => val.toFixed(2)

      return cents > 99 ? `$${toFixed(cents / 100)}` : `¢${cents}`
    },
    id() {
      return this.$route.query.id
    },
    user() {
      return this.$store.state.user || {}
    },

    disableQuantity() {
      return !this.requiredCoin
    },

    insufficientDeposit() {
      const totalCost = parseFloat(this.product.cost) * this.quantity

      return totalCost > parseFloat(this.user.deposit)
    },

    disablePurchase() {
      return this.soldOut || this.insufficientDeposit || this.isSeller
    },
    isSeller() {
      return this.user.role == 'seller'
    },
    purchaseText() {
      if (this.isSeller) {
        return 'Buyers only!'
      }

      if (this.product.ownerDeleted) {
        return 'Owner deleted'
      }

      if (this.soldOut) {
        return 'Sold out'
      }

      const text = this.insufficientDeposit ? `Insufficient coins` : 'Purchase'
      return this.purchasing ? '' : text
    },
    showDeposit() {
      return (
        !this.disablePurchase &&
        !this.isSeller &&
        !this.soldOut &&
        !this.ownerDeleted
      )
    }
  },

  async created() {
    this.loading = !this.product.id
    await this.fetchProduct()
  },

  methods: {
    ...mapActions(['getProduct']),

    async fetchProduct() {
      await this.getProduct(this.id)

      this.loading = false

      this.$commit('UPDATE', {
        path: 'productName',
        value: this.product.productName
      })
    },
    async purchase() {
      if (this.disablePurchase || this.isSeller) {
        return null
      }

      // reset dashboard's processing text;
      this.$commit('UPDATE', {
        path: 'processingDone',
        value: null
      })

      await this.$nextTick()

      // show button spinner
      this.purchasing = true

      const { data, error } = await buy.call(this, {
        id: this.id,
        amount: this.quantity
      })

      // hide spinner;
      this.purchasing = false

      // open processing modal
      this.$commit('UPDATE', {
        path: 'dashboardProcessing',
        value: true
      })

      await this.$nextTick()

      // update processingText;

      this.$commit('UPDATE', {
        path: 'processingDone',
        value: {
          title: error ? 'An error occured' : 'Purchase successful',
          subtitle: error?.message || data?.message,
          error: !!error,
          key: Date.now(),
          change: data?.change,
          changeTotal: data?.changeTotal
        }
      })

      // reset quantity;

      this.quantity = 1

      await this.$refreshUser()

      await this.fetchProduct()
    }
  }
}
</script>

<style></style>
