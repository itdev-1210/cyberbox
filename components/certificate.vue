<template>
    <div class="certificate__item collection__item">
        <img src="/more.png" alt="more" class="collection__item-more" @click="openModal(certificate.id)" v-if="owner">
		<div class="collection__item-modal" @mouseleave="modalId = 0" v-if="modalId === certificate.id">
			<div class="collection__item-modal-button" @click="routeCertificate">
				<img src="/outline-sell.svg" alt="sell">
				<h3>{{ visitMenuName }}</h3>
			</div>
			<div class="collection__item-modal-button" @click="showTransferModal = true" v-if="owner">
				<img src="/transfer-black.svg" alt="transfer">
				<h3>Transfer</h3>
			</div>
			<div class="collection__item-modal-button" @click="copyLink">
				<img src="/copy-link.svg" alt="copy">
				<h3>Copy link</h3>
			</div>
		</div>
        <div class="collection__item-image">
            <img :src="certificate.image" alt="item">
        </div>
        <div class="collection__item-info">
            <h2 class="collection__item-info-name">{{ certificateName }}</h2>
            <p class="collection__item-info-type">Price</p>
            <div class="collection__item-info-price" v-if="certificate.price">
                <img src="/celo.svg" alt="celo">
                <h3 class="collection__item-info-price-text">{{ certificate.price }}</h3>
            </div>
            <p class="certificate__item-description" :class="{ 'not-sale': owner || !saleAvailable }" v-else>{{ certificateDescription }}</p>
            <button class="collection__item-info-details" @click="routeCertificate" v-if="owner">Details</button>
            <button class="collection__item-info-details buy" @click="routeCertificate" v-else-if="buyAvailable">Buy</button>
            <button class="collection__item-info-details offset" @click="routeCertificate" v-else-if="saleAvailable">Offset now</button>
        </div>
		<transfer :nft="certificate" @closeModal="showTransferModal=false" v-if="showTransferModal" />
    </div>    
</template>

<script>
import transfer from '@/components/modals/transfer'
export default {
  props: ['certificate'],
  components: {
	transfer
  },
  data() {
	return {
	  modalId: 0,
	  showTransferModal: false
	}
  },
  computed: {
	certificateName() {
	  return this.getCertificateName(this.certificate, false)
	},
	visitMenuName() {
	  if (this.certificate.market_status === 'LISTED') {
        return 'Remove from sale'
      } else {
        return 'Sell'
      }
	},
    owner() {
      return this.$store.state.fullAddress === this.certificate.owner
	},
	priceVisible() {
      return this.certificate.price > 0
	},
	buyAvailable() {
      return !this.owner && this.certificate.price > 0
	},
	saleAvailable() {
	  return this.certificate.offset
	},
    certificateDescription() {
      return this.owner || !this.saleAvailable ? 'Not for sale' : '-'
    }
  },
  methods: {
	openModal(id) {
      if (this.modalId !== id) {
        this.modalId = id
      } else {
        this.modalId = 0
      }
    },
	copyLink() {
      const collectionUrl = location.protocol + '//' + location.host
      this.$copyText(`${collectionUrl}/collections/${this.certificate.contract}/${this.certificate.contract_id}`)
      this.$store.commit('setMessage', 'Link copied!')
      setTimeout(() => {
        this.$store.commit('setMessage', '')
      }, 2000)
    },
    routeCertificate() {
	  if (this.certificate.offset) {
		this.$router.push('/lending')
	  } else {
		this.$router.push(`/collections/CBCN/${this.certificate.contract_id}`)
	  }
    }
  }
}
</script>

<style lang="scss" scoped>
.certificate__item {
  height: 39.7rem;
  .collection__item-info-name {
	font-size: 1.8rem;
    padding-bottom: 4rem;
    border-bottom: .1rem solid $modalColor;
  }
  .collection__item-info-type {
	padding-top: 1.2rem;
	font-size: 1.2rem;
  }
  .collection__item-image {
    display: flex;
    align-items: center;
    justify-content: center;
    background: linear-gradient(0deg, rgba(255, 255, 255, 0.75), rgba(255, 255, 255, 0.75)), linear-gradient(46.74deg, #365BE0 -17.17%, #D676CF 48.99%, #FFE884 113%);
	max-width: 20rem;
	border-radius: 0.4rem;
	overflow: hidden;
    img {
      max-width: 100%;
    }
  }
  &-description {
	padding-top: 0.4rem;
    font-size: 1.4rem;
    &.not-sale {
      color: $border;
    }
  }
  .collection__item-info-details {
	margin-top: 1.6rem;
    &.buy {
      background: $white;
      border: 1px solid $modalColor;
    }
    &.offset {
      background: linear-gradient(90deg, #365BE0 -14.25%, #D676CF 48.65%, #FFE884 109.5%);
      color: $white;
      &.disabled {
        background: $white;
        pointer-events: none;
        border: 1px solid $modalColor;
        color: $border2;
      }
    }
  }

  @media (max-width: 460px) {
	height: 28.8rem;
	.collection__item-info-name {
	  padding-bottom: 3.2rem;
	  font-size: 1.4rem;
	}
  }
}
</style>