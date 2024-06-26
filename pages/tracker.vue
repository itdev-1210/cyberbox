<template>
	<section id="carbon">
		<div class="carbon">
			<div class="carbon__tracker">
				<img class="carbon__tracker-bg" :src="trackerBGImage" alt="background">
				<h2 class="carbon__title">Carbon offset tracker</h2>
				<p class="carbon__address" v-show="linkShared">by {{ walletAddress }}</p>
				<p class="carbon__tracker-year" :class="{shared: linkShared}">2022</p>
				<!-- <CustomSelect class="carbon__tracker-picker" :options="dateOptions" @change="filterByYear"/> -->
				<div class="carbon__tracker-info">
					<h2 class="carbon__tracker-info-title">{{ formatCO2(totalCO2Offset) }}</h2>
					<p class="carbon__tracker-info-description">Total offset (tCO2) <img class="carbon__tracker-info-description-img" :src="getCDNImage('plant.svg')" alt="plant"></p>
					<div class="carbon__tracker-info-block">
						<div class="carbon__tracker-info-block-item">
							<p class="carbon__tracker-info-block-item-value">{{ formatCO2(totalCertCO2) }}</p>
							<p class="carbon__tracker-info-block-item-name">ReFi Calendar<br/>(tCO2)</p>
						</div>
						<div class="carbon__tracker-info-block-item">
							<p class="carbon__tracker-info-block-item-value">{{ formatCO2(totalTradingCO2) }}</p>
							<p class="carbon__tracker-info-block-item-name">Trading<br/>(tCO2)</p>
						</div>
					</div>
				</div>
				<div class="carbon__tracker-buttons">
					<button class="carbon__tracker-buttons-button buy gradient-button" @click="gotoLending">Mint ReFi Calendar</button>
					<button class="carbon__tracker-buttons-button bonus" @click="showExchangeBonus=true" v-if="!bonusPurchased">Get a Bonus</button>
				</div>
				<div class="carbon__tracker-share">
					<a class="carbon__tracker-share-profile" :href="sharedCollectionUrl" v-show="linkShared">
						<img class="carbon__tracker-share-profile-img" :src="getCDNImage('earth-small.svg')" alt="earth">
						<span class="carbon__tracker-share-profile-name">Profile</span>
					</a>
					<ShareFrame class="carbon__tracker-share-frame"/>
				</div>
			</div>
			<div class="carbon__certificates">
				<h2 class="carbon__title">ReFi Calendar Collection</h2>
				<!-- <div class="carbon__certificates-header">
					<CustomSwitch class="carbon__certificates-header-switch" label="My Certificates" :value="myCertificate" @onChange="changeMyCertificateStatus"/>
				</div> -->
				<div class="carbon__certificates-list">
					<certificate :certificate="certificate" :sharedWallet="sharedWallet" :key="idx" v-for="(certificate, idx) of filteredCertificates"/>
				</div>
			</div>
			<ExchangeBonus @closeModal="showExchangeBonus = false" :bonusAvailable="bonusAvailable" @onExchange="showExchangeTokenModal" v-if="showExchangeBonus"/>
			<ExchangeToken @closeModal="showExchangeToken = false" v-if="showExchangeToken"/>
			<SuccessfullBuy v-if="showSuccessModal" :image="bonusImage" :name="certificateName" :certificate="true"/>
		</div>
	</section>
</template>

<script>
import { CERTIFICATE_TOKEN_TYPE } from '@/config'
import CustomSelect from '@/components/utility/CustomSelect.vue'
import ShareFrame from '@/components/ShareFrame.vue'
import certificate from '@/components/certificate.vue'
import ExchangeBonus from '@/components/modals/exchangeBonus.vue'
import ExchangeToken from '@/components/modals/exchangeToken.vue'
import SuccessfullBuy from '@/components/modals/successBuy'

const MAX_TON_AMOUNT = 20

export default {
  components: {
	CustomSelect,
	ShareFrame,
	certificate,
	ExchangeBonus,
	ExchangeToken,
	SuccessfullBuy
  },
  data() {
	return {
	  myCertificate: false,
	  certificateList: [],
	  filteredCertificates: [],
	  yearFilter: 2022,
	  progressSize: 0,
	  trackingInfo: {},
	  totalCertCO2: 0,
	  totalTradingCO2: 0,
	  totalCO2Offset: 0,
	  bonusAvailable: false,
	  showShareFrame: false,
	  showExchangeBonus: false,
	  showExchangeToken: false,
	  bonusPurchased: false,
	  webVersion: false,
	  linkShared: false
	}
  },
  computed: {
	dateOptions() {
	  return [
		{ key: '2022', value: '2022' }
	  ]
	},
	certificateDateOptions() {
	  return [
		{ key: '2022', value: '2022' },
	  ]
	},
	address() {
	  return this.$store.state.address
	},
	sharedWallet() {
	  return this.$store.state.sharedWallet
	},
	sharedCollectionUrl() {
	  return `/mycollection?wallet=${this.sharedWallet}`
	},
	walletAddress() {
	  const address = this.sharedWallet || this.address
      if (address) {
        const startID = address.split("").slice(0, 6);
        const endID = address.split("").slice(-4);
        const dotArr = [".", ".", "."];
        return startID
          .concat(dotArr)
          .concat(endID)
          .join("");
	  }
	  return ''
	},
	trackerBGImage() {
	  return this.webVersion ? this.getCDNImage('tracker-bg.webp') : this.getCDNImage('tracker-bg-mobile.webp')
	},
	ownedCertificates() {
	  return this.$store.state.certificateList
	},
	saleCertificates() {
	  return this.$store.state.certificateSaleList
	},
	showSuccessModal() {
      return this.$store.state.successBuyToken
	},
	bonusImage() {
	  return this.getCDNImage('certificates/rare.webp')
	},
	certificateName() {
	  return 'Rare 2022'
	},
	refiPrice() {
	  return this.$store.state.cMCO2Price
	}
  },
  async created() {
	this.$nextTick(function() {
	  this.webVersion = !this.isMobile()
	})
	this.updateSharedWallet()
	this.$store.commit('changeSuccessBuyToken', false)
	this.yearFilter = new Date().getFullYear()
	if (process.broswer) {
	  if (!this.isMobie()) {
		this.progressSize = Math.round(0.6945 * document.body.offsetHeight / 1000)
	  } else {
		this.progressSize = Math.round(3.125 * document.body.offsetHeight / 1000)
	  }
	}
	if (this.$route.query.mycert) {
	  this.myCertificate = true
	}

	this.reloadPageData()
  },
  mounted() {
	this.updateCertificateList()
	if (this.$refs.trackerProgress) {
	  this.progressSize = this.$refs.trackerProgress.offsetWidth
	}
  },
  watch: {
	$route(from, to) {
	  if (to !== from) {
		this.updateSharedWallet(true)
	  }
    },
	address() {
	  if (this.$store.state.address) {
		this.$store.dispatch('checkBuyTokenApproved', 1.0)
		if (this.filteredCertificates.length === 0) {
		  this.updateCertificateList()
		}
	  }
	},
	ownedCertificates() {
	  this.updateCertificateList()
	}
  },
  head() {
    return {
      meta: [
				{ hid: 'og:image', property: 'og:image', content: this.getCDNImage('tracker-banner.png') },
				{ hid: 'twitter:image', name: 'twitter:image', content: this.getCDNImage('tracker-banner.png') }
      ]
    }
  },
  methods: {
	formatCO2(value) {
		if (value === 0) {
			return '0.0'
		} else {
			const diff = value - Math.floor(value)
			const floatingLen = diff.toString().length - 2
			if (floatingLen > 1) {
				return value >= 0.01 ? value.toFixed(2) : '<0.01'
			} else {
				return value >= 0.01 ? value.toFixed(1) : '<0.01'
			}
		}
	},
	async reloadPageData() {
	  const trackingInfo = await this.$store.dispatch('getCarbonData')
	  this.totalTradingCO2 = trackingInfo.totalTradingCelo
	  this.totalCertCO2 = trackingInfo.totalCount
	  this.totalCO2Offset = parseFloat(this.totalCertCO2.toFixed(1)) + parseFloat(this.totalTradingCO2.toFixed(1))
	  this.$store.dispatch('getCertificates')
	},
	updateSharedWallet(routeChanged = false) {
	  const oldLinkShared = this.linkShared
	  if (this.$route.query.wallet) {
		this.linkShared = true
		this.$store.commit('setSharedWallet', this.$route.query.wallet)
	  } else {
		this.linkShared = false
		this.$store.commit('setSharedWallet', null)
	  }
	  if (routeChanged && this.linkShared !== oldLinkShared) {
		this.reloadPageData()
	  }
	},
	updateCertificateList() {
	  if (this.$store.state.address) {
		let newList = this.getCertificatesOfYear(this.yearFilter)
		const additionalList = []
		const date = new Date()
		const currYear = date.getFullYear()
		const currMonth = date.getMonth() + 1
		const bonusNft = this.ownedCertificates.find(item => item.year === currYear && item.token_type === CERTIFICATE_TOKEN_TYPE.BONUS)
		this.bonusPurchased = !!bonusNft
		const invisibleList = []
		newList.forEach((item, index) => {
		  const foundIndex = this.ownedCertificates.findIndex(oItem => oItem.year === item.year && oItem.month === item.month)
		  if (foundIndex >= 0) {
			newList[index] = {
			  ...this.ownedCertificates[foundIndex],
			  image: this.getCertificateImage(newList[index])
			}
		  } else {
			if (newList[index].year !== currYear || (newList[index].year === currYear && newList[index].month < currMonth)) {
			  invisibleList.push(item)
			}
			// if (newList[index].year === currYear && newList[index].month < currMonth) {
			//   const foundSaleIndex = this.saleCertificates.findIndex(oItem => oItem.year === item.year && oItem.month === item.month && oItem.year === currYear && oItem.month !== currMonth)
			//   if (foundSaleIndex >= 0) {
			// 	newList[index].offset = false
			// 	newList[index].contract_id = this.saleCertificates[foundSaleIndex].contract_id
			// 	newList[index].price = this.saleCertificates[foundSaleIndex].price
			//   }
			// }
		  }
		})
		newList = newList.filter(item => !invisibleList.find(invItem => item.year === invItem.year && item.month === invItem.month))
		const ownedCount = this.ownedCertificates.filter(item => item.year === currYear && item.token_type === CERTIFICATE_TOKEN_TYPE.MONTH).length
		this.bonusAvailable = ownedCount === 12 && !bonusNft
		this.ownedCertificates.forEach(oItem => {
		  const foundIndex = newList.findIndex(item => oItem.year === item.year && oItem.month === item.month)
		  if (foundIndex < 0) {
			newList.push({
			  ...oItem,
			  image: this.getCertificateImage(oItem)
			})
		  }
		})
		newList = newList.sort((a, b) => ((a.year + a.month) - (b.year + b.month)))
		this.certificateList = newList
		this.changeFilter()
	  }
	},
	changeFilter() {
	  if (this.myCertificate) {
		this.filteredCertificates = this.ownedCertificates.filter(item => item.year === this.yearFilter)
	  } else {
		this.filteredCertificates = this.certificateList
	  }
	},
	changeMyCertificateStatus() {
	  this.myCertificate = !this.myCertificate
	  this.changeFilter()
	},
	showExchangeTokenModal() {
	  this.showExchangeBonus = false
	  this.showExchangeToken = true
	},
	filterByYear(yearStr) {
	  let newCertifictes = []
	  const year = parseInt(yearStr)
	  this.yearFilter = year
	  this.certificateList = this.getCertificatesOfYear(year)
	  this.changeFilter()
	},
	gotoLending() {
	  this.sendEvent({
		category: 'Browse',
		eventName: 'minter_enter',
		properties: {
		  minter_enter: 'Tracker_button'
		}
	  })
	  this.$router.push('/calendar')
	}
  }
}
</script>

<style lang="scss" scoped>
.carbon {
  &__title {
	font-family: Cabin-Medium;
	font-weight: 500;
	font-size: 3.2rem;
	text-align: center;
  }
  &__address {
	margin-top: 1rem;
	text-align: center;
	font-weight: 600;
	font-size: 1.6rem;	  
  }
  &__tracker {
	display: flex;
	flex-direction: column;
	align-items: center;
	padding-top: 4.2rem;
  	padding-bottom: 3.4rem;
	background: rgba(255, 255, 255, 0.2);
	position: relative;
	&-bg {
	  position: absolute;
	  left: 0;
	  top: 0;
	  width: 100%;
	  height: 100%;
	  z-index: -1;
	}
	&-share {
	  display: flex;
	  align-items: center;
	  position: absolute;
	  top: 4.1rem;
	  right: 6rem;
	  &-profile {
		display: flex;
		align-items: center;
		margin-right: 2rem;
		&-img {
		  width: 2.2rem;
		  margin-right: 0.8rem;
		}
		&-name {
		  font-weight: 600;
		  font-size: 1.8rem;
		  color: $textColor3;
		}
	  }
	}
	&-year {
	  margin: 1.8rem auto 0;
	  text-align: center;
	  font-weight: 600;
	  font-size: 1.6rem;
	  color: $grayDark;
	}
	&-info {
	  background: $white;
	  width: 34rem;
	  margin-top: 3.2rem;
	  padding: 2.4rem 2.2rem;
	  border-radius: 2.4rem;
	  text-align: center;
	  &-title {
		font-weight: 500;
		font-size: 6.2rem;
		line-height: 5.6rem;
	  }
	  &-description {
		margin-top: 0.8rem;
		font-weight: 600;
		font-size: 1.4rem;
		color: $grayLight;
		img {
		  width: 1.5rem;
		}
	  }
	  &-block {
		display: flex;
		justify-content: space-between;
		margin-top: 2rem;
		padding: 2.2rem 3.4rem 0;
		border-top: 1px solid $modalColor;
		&-item {
		  text-align: center;
		  &-value {
			font-weight: 600;
			font-size: 1.8rem;
		  }
		  &-name {
			margin-top: 0.8rem;
			text-align: center;
			font-weight: 600;
			font-size: 1.2rem;
			color: $grayLight;
		  }
		}
	  }
	}
	&-buttons {
	  display: flex;
	  margin-top: 3.8rem;
	  &-button {
		width: 18.5rem;
		height: 3.6rem;
		padding: 0.8rem 2.4rem;
		border-radius: 2.3rem;
		text-align: center;
		font-weight: 600;
		font-size: 1.4rem;
		background: transparent;
		white-space: nowrap;
		&.buy {
		  color: $white;
		  &::after {
			background: linear-gradient(to right, #365BE0, #D676CF, #FFE884);
		  }
		}
		&.bonus {
		  margin-left: 0.8rem;
		  border: 1px solid $border;
		  color: $textColor3;
		}
	  }
	}
  }
  &__certificates {
	padding: 5.8rem 6rem 0;
	.carbon__title {
	  padding-bottom: 0;
	}
	&-header {
	  display: flex;
	  justify-content: flex-end;
	  &-switch {
		::v-deep .custom-switch-text {
		  font-weight: 400;
		}
	  }
	}
	&-list {
	  display: grid;
	  grid-template-columns: repeat(6, 20rem);
	  grid-column-gap: 2.4rem;
	  grid-row-gap: 3.5rem;
	  padding-top: 5.4rem;
	  padding-bottom: 2.4rem;
	}
  }
  @media (max-width: 460px) {
	&__title {
	  font-family: OpenSans-SemiBold;
	  font-weight: 600;
	  font-size: 1.8rem;
	}
	&__address {
	  font-size: 1.4rem;
	}
	&__tracker {
	  padding-top: 7.3rem;
	  padding-bottom: 3.6rem;
	  &-share {
		top: 1.6rem;
		right: 1rem;
	  }
	  &-year {
		font-size: 1.4rem;
		&.shared {
		  margin-top: 0.8rem;
		}
	  }
	  &-info {
		width: calc(100% - 6rem);
		&-block {
		  &-item {
			&-name {
			  font-size: 1rem;
			}
		  }
		}
		&-title {
		  font-size: 4.2rem;
		}
	  }
	  &-buttons {
		flex-direction: column;
		&-button {
		  &.buy {
			padding-left: 1rem;
			padding-right: 1rem;
		  }
		  &.bonus {
			margin-left: 0;
			margin-top: 1.2rem;
		  }
		}
	  }
	}
	&__certificates {
	  padding: 3.2rem 0.8rem 0;
	  .carbon__title {
		font-size: 1.6rem;
	  }
	  &-list {
		padding-top: 4rem;
		grid-template-columns: repeat(2, 14.4rem);
		grid-column-gap: 1.6rem;
		grid-row-gap: 1.6rem;
	  }
	}
  }
}
</style>
