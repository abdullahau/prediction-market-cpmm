Here are some relavent codes from Manifold's github:

## math.ts

```
import { sortBy, sum } from 'lodash'

export const logInterpolation = (min: number, max: number, value: number) => {
  if (value <= min) return 0
  if (value >= max) return 1

  return Math.log(value - min + 1) / Math.log(max - min + 1)
}

export const logit = (x: number) => Math.log(x / (1 - x))

export function median(xs: number[]) {
  if (xs.length === 0) return NaN

  const sorted = sortBy(xs, (x) => x)
  const mid = Math.floor(sorted.length / 2)
  if (sorted.length % 2 === 0) {
    return (sorted[mid - 1] + sorted[mid]) / 2
  }
  return sorted[mid]
}

export function average(xs: number[]) {
  return xs.length === 0 ? 0 : sum(xs) / xs.length
}

export function sumOfSquaredError(xs: number[]) {
  const mean = average(xs)
  let total = 0
  for (const x of xs) {
    const error = x - mean
    total += error * error
  }
  return total
}

export const EPSILON = 0.00000001

export function floatingEqual(a: number, b: number, epsilon = EPSILON) {
  return Math.abs(a - b) < epsilon
}
export function floatingGreater(a: number, b: number, epsilon = EPSILON) {
  return a - epsilon > b
}

export function floatingGreaterEqual(a: number, b: number, epsilon = EPSILON) {
  return a + epsilon >= b
}

export function floatingLesserEqual(a: number, b: number, epsilon = EPSILON) {
  return a - epsilon <= b
}```

## matrix.ts

```
import { max, sumBy } from 'lodash'

// each row has [column, value] pairs
type SparseMatrix = [number, number][][]

// Code originally from: https://github.com/johnpaulada/matrix-factorization-js/blob/master/src/matrix-factorization.js
// Used to implement recommendations through collaborative filtering: https://towardsdatascience.com/recommender-systems-matrix-factorization-using-pytorch-bd52f46aa199
// See also: https://en.wikipedia.org/wiki/Matrix_factorization_(recommender_systems)

/**
 * Gets the factors of a sparse matrix
 *
 * @param TARGET_MATRIX target matrix, where each row specifies a subset of all columns.
 * @param FEATURES Number of latent features
 * @param ITERS Number of times to move towards the real factors
 * @param LEARNING_RATE Learning rate
 * @param REGULARIZATION_RATE Regularization amount, i.e. amount of bias reduction
 * @returns An array containing the two factor matrices
 */
export function factorizeMatrix(
  TARGET_MATRIX: SparseMatrix,
  FEATURES = 5,
  ITERS = 5000,
  LEARNING_RATE = 0.0002,
  REGULARIZATION_RATE = 0.02,
  THRESHOLD = 0.001
) {
  const initCell = () => (2 * Math.random()) / FEATURES
  const m = TARGET_MATRIX.length
  const n = (max(TARGET_MATRIX.flatMap((r) => r.map(([j]) => j))) ?? -1) + 1
  const points = sumBy(TARGET_MATRIX, (r) => r.length)
  const mFeatures = fillMatrix(m, FEATURES, initCell)
  const nFeatures = fillMatrix(n, FEATURES, initCell)

  console.log('rows', m, 'columns', n, 'numPoints', points)

  const updateFeature = (a: number, b: number, error: number) =>
    a + LEARNING_RATE * (2 * error * b - REGULARIZATION_RATE * a)

  const dotProduct = (i: number, j: number) => {
    let result = 0
    for (let k = 0; k < FEATURES; k++) {
      result += mFeatures[i * FEATURES + k] * nFeatures[j * FEATURES + k]
    }
    return result
  }

  // Iteratively figure out correct factors.
  for (let iter = 0; iter < ITERS; iter++) {
    for (let i = 0; i < m; i++) {
      for (const [j, targetValue] of TARGET_MATRIX[i]) {
        // to approximate the value for target_ij, we take the dot product of the features for m[i] and n[j]
        const error = targetValue - dotProduct(i, j)
        // update factor matrices
        for (let k = 0; k < FEATURES; k++) {
          const a = mFeatures[i * FEATURES + k]
          const b = nFeatures[j * FEATURES + k]
          mFeatures[i * FEATURES + k] = updateFeature(a, b, error)
          nFeatures[j * FEATURES + k] = updateFeature(b, a, error)
        }
      }
    }

    if (iter % 50 === 0 || iter === ITERS - 1) {
      let totalError = 0
      for (let i = 0; i < m; i++) {
        for (const [j, targetValue] of TARGET_MATRIX[i]) {
          // add up squared error of current approximated value
          totalError += (targetValue - dotProduct(i, j)) ** 2
          // mqp: idk what this part of the error means lol
          for (let k = 0; k < FEATURES; k++) {
            const a = mFeatures[i * FEATURES + k]
            const b = nFeatures[j * FEATURES + k]
            totalError += (REGULARIZATION_RATE / 2) * (a ** 2 + b ** 2)
          }
        }
      }
      console.log(iter, 'error', totalError / points)

      // Complete factorization process if total error falls below a certain threshold
      if (totalError / points < THRESHOLD) break
    }
  }

  return [mFeatures, nFeatures, dotProduct] as const
}

/**
 * Creates an m x n matrix filled with the result of given fill function.
 */
function fillMatrix(m: number, n: number, fill: () => number) {
  const matrix = new Float64Array(m * n)
  for (let i = 0; i < m; i++) {
    for (let j = 0; j < n; j++) {
      matrix[i * n + j] = fill()
    }
  }
  return matrix
}```


## bets.ts

```
import { groupBy, mapValues } from 'lodash'
import { Fees } from './fees'
import { maxMinBin } from './chart'

/************************************************

supabase status: columns exist for
  userId: text
  createdTime: timestamp (from millis)
  amount: number
  shares: number
  outcome: text
  probBefore: number
  probAfter: number
  isRedemption: boolean
  visibility: text

*************************************************/

export type Bet = {
  id: string
  userId: string

  contractId: string
  answerId?: string // For multi-binary contracts
  createdTime: number
  updatedTime?: number // Generated on supabase, useful for limit orders

  amount: number // bet size; negative if SELL bet
  loanAmount?: number
  outcome: string
  shares: number // dynamic parimutuel pool weight or fixed ; negative if SELL bet

  probBefore: number
  probAfter: number

  fees: Fees

  isApi?: boolean // true if bet was placed via API

  isRedemption: boolean
  /** @deprecated */
  challengeSlug?: string

  replyToCommentId?: string
  betGroupId?: string // Used to group buys on MC sumsToOne contracts
} & Partial<LimitProps>

export type NumericBet = Bet & {
  value: number
  allOutcomeShares: { [outcome: string]: number }
  allBetAmounts: { [outcome: string]: number }
}

// Binary market limit order.
export type LimitBet = Bet & LimitProps

type LimitProps = {
  orderAmount: number // Amount of mana in the order
  limitProb: number // [0, 1]. Bet to this probability.
  isFilled: boolean // Whether all of the bet amount has been filled.
  isCancelled: boolean // Whether to prevent any further fills.
  // A record of each transaction that partially (or fully) fills the orderAmount.
  // I.e. A limit order could be filled by partially matching with several bets.
  // Non-limit orders can also be filled by matching with multiple limit orders.
  fills: fill[]
  expiresAt?: number // ms since epoch.
}

export type fill = {
  // The id the bet matched against, or null if the bet was matched by the pool.
  matchedBetId: string | null
  amount: number
  shares: number
  timestamp: number
  // Note: Old fills might have no fees, and the value would be undefined.
  fees: Fees
  // If the fill is a sale, it means the matching bet has shares of the same outcome.
  // I.e. -fill.shares === matchedBet.shares
  isSale?: boolean
}

export const calculateMultiBets = (
  betPoints: {
    x: number
    y: number
    answerId: string
  }[]
) => {
  return mapValues(groupBy(betPoints, 'answerId'), (bets) =>
    maxMinBin(
      bets.sort((a, b) => a.x - b.x),
      500
    )
  )
}
export type maker = {
  bet: LimitBet
  amount: number
  shares: number
  timestamp: number
}
```

## fees.ts

```
import { addObjects } from 'common/util/object'
import { TWOMBA_ENABLED } from './envs/constants'

export const FEE_START_TIME = 1713292320000

const TAKER_FEE_CONSTANT = 0.07
export const getTakerFee = (shares: number, prob: number) => {
  return TAKER_FEE_CONSTANT * prob * (1 - prob) * shares
}

const CREATORS_EARN_WHOLE_FEE_UP_TO = 1000
export const getFeesSplit = (
  totalFees: number,
  previouslyCollectedFees: Fees
) => {
  if (TWOMBA_ENABLED) {
    return {
      creatorFee: 0,
      platformFee: totalFees,
      liquidityFee: 0,
    }
  }

  const before1k = Math.max(
    0,
    CREATORS_EARN_WHOLE_FEE_UP_TO - previouslyCollectedFees.creatorFee
  )
  const allToCreatorAmount = Math.min(totalFees, before1k)
  const splitWithCreatorAmount = totalFees - allToCreatorAmount
  return {
    creatorFee: allToCreatorAmount + splitWithCreatorAmount * 0.5,
    platformFee: splitWithCreatorAmount * 0.5,
    liquidityFee: 0,
  }
}

export const FLAT_TRADE_FEE = 0.1
export const FLAT_COMMENT_FEE = 1

export const DPM_PLATFORM_FEE = 0.0
export const DPM_CREATOR_FEE = 0.0
export const DPM_FEES = DPM_PLATFORM_FEE + DPM_CREATOR_FEE

export type Fees = {
  creatorFee: number
  platformFee: number
  liquidityFee: number
}

export const noFees: Fees = {
  creatorFee: 0,
  platformFee: 0,
  liquidityFee: 0,
}

export const getFeeTotal = (fees: Fees) => {
  return fees.creatorFee + fees.platformFee + fees.liquidityFee
}

export const sumAllFees = (fees: Fees[]) => {
  let totalFees = noFees
  fees.forEach((totalFee) => (totalFees = addObjects(totalFees, totalFee)))
  return getFeeTotal(totalFees)
}
```

## calculate-cpmm.ts

```
import { groupBy, mapValues, sumBy } from 'lodash'
import { LimitBet } from './bet'

import { Fees, getFeesSplit, getTakerFee, noFees } from './fees'
import { LiquidityProvision } from './liquidity-provision'
import { computeFills } from './new-bet'
import { binarySearch } from './util/algos'
import { EPSILON, floatingEqual } from './util/math'
import {
  calculateCpmmMultiArbitrageSellNo,
  calculateCpmmMultiArbitrageSellYes,
} from './calculate-cpmm-arbitrage'
import { Answer } from './answer'
import { CPMMContract, CPMMMultiContract } from 'common/contract'

export type CpmmState = {
  pool: { [outcome: string]: number }
  p: number
  collectedFees: Fees
}

export function getCpmmProbability(
  pool: { [outcome: string]: number },
  p: number
) {
  const { YES, NO } = pool
  return (p * NO) / ((1 - p) * YES + p * NO)
}

export function getCpmmProbabilityAfterBetBeforeFees(
  state: CpmmState,
  outcome: string,
  bet: number
) {
  const { pool, p } = state
  const shares = calculateCpmmShares(pool, p, bet, outcome)
  const { YES: y, NO: n } = pool

  const [newY, newN] =
    outcome === 'YES'
      ? [y - shares + bet, n + bet]
      : [y + bet, n - shares + bet]

  return getCpmmProbability({ YES: newY, NO: newN }, p)
}

export function getCpmmOutcomeProbabilityAfterBet(
  state: CpmmState,
  outcome: string,
  bet: number
) {
  const { newPool } = calculateCpmmPurchase(state, bet, outcome)
  const p = getCpmmProbability(newPool, state.p)
  return outcome === 'NO' ? 1 - p : p
}

// before liquidity fee
export function calculateCpmmShares(
  pool: {
    [outcome: string]: number
  },
  p: number,
  betAmount: number,
  betChoice: string
) {
  if (betAmount === 0) return 0

  const { YES: y, NO: n } = pool
  const k = y ** p * n ** (1 - p)

  return betChoice === 'YES'
    ? // https://www.wolframalpha.com/input?i=%28y%2Bb-s%29%5E%28p%29*%28n%2Bb%29%5E%281-p%29+%3D+k%2C+solve+s
      y + betAmount - (k * (betAmount + n) ** (p - 1)) ** (1 / p)
    : n + betAmount - (k * (betAmount + y) ** -p) ** (1 / (1 - p))
}

export function getCpmmFees(
  state: CpmmState,
  betAmount: number,
  outcome: string
) {
  // Do a few iterations toward average probability of the bet minus fees.
  // Charging fees means the bet amount is lower and the average probability moves slightly less far.
  let fee = 0
  for (let i = 0; i < 10; i++) {
    const betAmountAfterFee = betAmount - fee
    const shares = calculateCpmmShares(
      state.pool,
      state.p,
      betAmountAfterFee,
      outcome
    )
    const averageProb = betAmountAfterFee / shares
    fee = getTakerFee(shares, averageProb)
  }

  const totalFees = betAmount === 0 ? 0 : fee
  const fees = getFeesSplit(totalFees, state.collectedFees)

  const remainingBet = betAmount - totalFees

  return { remainingBet, totalFees, fees }
}

export function calculateCpmmSharesAfterFee(
  state: CpmmState,
  bet: number,
  outcome: string
) {
  const { pool, p } = state
  const { remainingBet } = getCpmmFees(state, bet, outcome)

  return calculateCpmmShares(pool, p, remainingBet, outcome)
}

export function calculateCpmmPurchase(
  state: CpmmState,
  bet: number,
  outcome: string,
  freeFees?: boolean
) {
  const { pool, p } = state
  const { remainingBet, fees } = freeFees
    ? {
        remainingBet: bet,
        fees: noFees,
      }
    : getCpmmFees(state, bet, outcome)

  const shares = calculateCpmmShares(pool, p, remainingBet, outcome)
  const { YES: y, NO: n } = pool

  const { liquidityFee: fee } = fees

  const [newY, newN] =
    outcome === 'YES'
      ? [y - shares + remainingBet + fee, n + remainingBet + fee]
      : [y + remainingBet + fee, n - shares + remainingBet + fee]

  const postBetPool = { YES: newY, NO: newN }

  const { newPool, newP } = addCpmmLiquidity(postBetPool, p, fee)

  return { shares, newPool, newP, fees }
}

export function calculateCpmmAmountToProb(
  state: CpmmState,
  prob: number,
  outcome: 'YES' | 'NO'
) {
  if (prob <= 0 || prob >= 1 || isNaN(prob)) return Infinity
  if (outcome === 'NO') prob = 1 - prob

  const { pool, p } = state
  const { YES: y, NO: n } = pool
  const k = y ** p * n ** (1 - p)
  return outcome === 'YES'
    ? // https://www.wolframalpha.com/input?i=-1+%2B+t+-+((-1+%2B+p)+t+(k%2F(n+%2B+b))^(1%2Fp))%2Fp+solve+b
      ((p * (prob - 1)) / ((p - 1) * prob)) ** -p *
        (k - n * ((p * (prob - 1)) / ((p - 1) * prob)) ** p)
    : (((1 - p) * (prob - 1)) / (-p * prob)) ** (p - 1) *
        (k - y * (((1 - p) * (prob - 1)) / (-p * prob)) ** (1 - p))
}

export function calculateCpmmAmountToProbIncludingFees(
  state: CpmmState,
  prob: number,
  outcome: 'YES' | 'NO'
) {
  const amount = calculateCpmmAmountToProb(state, prob, outcome)
  const shares = calculateCpmmShares(state.pool, state.p, amount, outcome)
  const averageProb = amount / shares
  const fees = getTakerFee(shares, averageProb)
  return amount + fees
}

export function calculateCpmmAmountToBuySharesFixedP(
  state: CpmmState,
  shares: number,
  outcome: 'YES' | 'NO'
) {
  if (!floatingEqual(state.p, 0.5)) {
    throw new Error(
      'calculateAmountToBuySharesFixedP only works for p = 0.5, got ' + state.p
    )
  }

  const { YES: y, NO: n } = state.pool
  if (outcome === 'YES') {
    // https://www.wolframalpha.com/input?i=%28y%2Bb-s%29%5E0.5+*+%28n%2Bb%29%5E0.5+%3D+y+%5E+0.5+*+n+%5E+0.5%2C+solve+b
    return (
      (shares - y - n + Math.sqrt(4 * n * shares + (y + n - shares) ** 2)) / 2
    )
  }
  return (
    (shares - y - n + Math.sqrt(4 * y * shares + (y + n - shares) ** 2)) / 2
  )
}

// Faster version assuming p = 0.5
export function calculateAmountToBuySharesFixedP(
  state: CpmmState,
  shares: number,
  outcome: 'YES' | 'NO',
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  freeFees?: boolean
) {
  const { takers } = computeFills(
    state,
    outcome,
    // First, bet more than required to get shares.
    shares,
    undefined,
    unfilledBets,
    balanceByUserId,
    undefined,
    freeFees
  )

  let currShares = 0
  let currAmount = 0
  for (const fill of takers) {
    const { amount: fillAmount, shares: fillShares, matchedBetId } = fill

    if (floatingEqual(currShares + fillShares, shares)) {
      return currAmount + fillAmount
    }
    if (currShares + fillShares > shares) {
      // This is first fill that goes over the required shares.
      if (matchedBetId) {
        // Match a portion of the fill to get the exact shares.
        const remainingShares = shares - currShares
        const remainingAmount = fillAmount * (remainingShares / fillShares)
        return currAmount + remainingAmount
      }
      // Last fill was from AMM. Break to compute the cpmmState at this point.
      break
    }

    currShares += fillShares
    currAmount += fillAmount
  }

  const remaningShares = shares - currShares

  // Recompute up to currAmount to get the current cpmmState.
  const { cpmmState } = computeFills(
    state,
    outcome,
    currAmount,
    undefined,
    unfilledBets,
    balanceByUserId,
    undefined,
    freeFees
  )
  const fillAmount = calculateCpmmAmountToBuySharesFixedP(
    cpmmState,
    remaningShares,
    outcome
  )
  const fillAmountFees = freeFees
    ? 0
    : getTakerFee(remaningShares, fillAmount / remaningShares)
  return currAmount + fillAmount + fillAmountFees
}

export function calculateCpmmMultiSumsToOneSale(
  answers: Answer[],
  answerToSell: Answer,
  shares: number,
  outcome: 'YES' | 'NO',
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  if (Math.round(shares) < 0) {
    throw new Error('Cannot sell non-positive shares')
  }

  const { newBetResult, otherBetResults } =
    outcome === 'YES'
      ? calculateCpmmMultiArbitrageSellYes(
          answers,
          answerToSell,
          shares,
          limitProb,
          unfilledBets,
          balanceByUserId,
          collectedFees
        )
      : calculateCpmmMultiArbitrageSellNo(
          answers,
          answerToSell,
          shares,
          limitProb,
          unfilledBets,
          balanceByUserId,
          collectedFees
        )

  const buyAmount = sumBy(newBetResult.takers, (taker) => taker.amount)
  // Transform buys of opposite outcome into sells.
  const saleTakers = newBetResult.takers.map((taker) => ({
    ...taker,
    // You bought opposite shares, which combine with existing shares, removing them.
    shares: -taker.shares,
    // Opposite shares combine with shares you are selling for Ṁ of shares.
    // You paid taker.amount for the opposite shares.
    // Take the negative because this is money you gain.
    amount: -(taker.shares - taker.amount),
    isSale: true,
  }))

  const saleValue = -sumBy(saleTakers, (taker) => taker.amount)

  const transformedNewBetResult = {
    ...newBetResult,
    takers: saleTakers,
    outcome,
  }

  return {
    saleValue,
    buyAmount,
    newBetResult: transformedNewBetResult,
    otherBetResults,
  }
}

export function calculateAmountToBuyShares(
  state: CpmmState,
  shares: number,
  outcome: 'YES' | 'NO',
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number }
) {
  const prob = getCpmmProbability(state.pool, state.p)
  const minAmount = shares * (outcome === 'YES' ? prob : 1 - prob)

  // Search for amount between bounds.
  // Min share price is based on current probability, and max is Ṁ1 each.
  return binarySearch(minAmount, shares, (amount) => {
    const { takers } = computeFills(
      state,
      outcome,
      amount,
      undefined,
      unfilledBets,
      balanceByUserId
    )

    const totalShares = sumBy(takers, (taker) => taker.shares)
    return totalShares - shares
  })
}

export function calculateCpmmAmountToBuyShares(
  contract: CPMMContract | CPMMMultiContract,
  shares: number,
  outcome: 'YES' | 'NO',
  allUnfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  answer?: Answer
) {
  const startCpmmState =
    contract.mechanism === 'cpmm-1'
      ? contract
      : {
          pool: { YES: answer!.poolYes, NO: answer!.poolNo },
          p: 0.5,
          collectedFees: contract.collectedFees,
        }

  const unfilledBets = answer?.id
    ? allUnfilledBets.filter((b) => b.answerId === answer.id)
    : allUnfilledBets

  if (contract.mechanism === 'cpmm-1') {
    return calculateAmountToBuyShares(
      startCpmmState,
      shares,
      outcome,
      unfilledBets,
      balanceByUserId
    )
  } else if (contract.mechanism === 'cpmm-multi-1') {
    return calculateAmountToBuySharesFixedP(
      startCpmmState,
      shares,
      outcome,
      unfilledBets,
      balanceByUserId
    )
  } else {
    throw new Error('Only works for cpmm-1 and cpmm-multi-1')
  }
}

export function calculateCpmmSale(
  state: CpmmState,
  shares: number,
  outcome: 'YES' | 'NO',
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number }
) {
  if (Math.round(shares) < 0) {
    throw new Error('Cannot sell non-positive shares')
  }

  const oppositeOutcome = outcome === 'YES' ? 'NO' : 'YES'
  const buyAmount = calculateAmountToBuyShares(
    state,
    shares,
    oppositeOutcome,
    unfilledBets,
    balanceByUserId
  )

  const { cpmmState, makers, takers, totalFees, ordersToCancel } = computeFills(
    state,
    oppositeOutcome,
    buyAmount,
    undefined,
    unfilledBets,
    balanceByUserId
  )

  // Transform buys of opposite outcome into sells.
  const saleTakers = takers.map((taker) => ({
    ...taker,
    // You bought opposite shares, which combine with existing shares, removing them.
    shares: -taker.shares,
    // Opposite shares combine with shares you are selling for Ṁ of shares.
    // You paid taker.amount for the opposite shares.
    // Take the negative because this is money you gain.
    amount: -(taker.shares - taker.amount),
    isSale: true,
  }))

  const saleValue = -sumBy(saleTakers, (taker) => taker.amount)

  return {
    saleValue,
    buyAmount,
    cpmmState,
    fees: totalFees,
    makers,
    takers: saleTakers,
    ordersToCancel,
  }
}

export function getCpmmProbabilityAfterSale(
  state: CpmmState,
  shares: number,
  outcome: 'YES' | 'NO',
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number }
) {
  const { cpmmState } = calculateCpmmSale(
    state,
    shares,
    outcome,
    unfilledBets,
    balanceByUserId
  )
  return getCpmmProbability(cpmmState.pool, cpmmState.p)
}

export function getCpmmLiquidity(
  pool: { [outcome: string]: number },
  p: number
) {
  const { YES, NO } = pool
  return YES ** p * NO ** (1 - p)
}

export function getMultiCpmmLiquidity(pool: { YES: number; NO: number }) {
  return getCpmmLiquidity(pool, 0.5)
}

export function addCpmmLiquidity(
  pool: { [outcome: string]: number },
  p: number,
  amount: number
) {
  const prob = getCpmmProbability(pool, p)

  //https://www.wolframalpha.com/input?i=p%28n%2Bb%29%2F%28%281-p%29%28y%2Bb%29%2Bp%28n%2Bb%29%29%3Dq%2C+solve+p
  const { YES: y, NO: n } = pool
  const numerator = prob * (amount + y)
  const denominator = amount - n * (prob - 1) + prob * y
  const newP = numerator / denominator

  const newPool = { YES: y + amount, NO: n + amount }

  const oldLiquidity = getCpmmLiquidity(pool, newP)
  const newLiquidity = getCpmmLiquidity(newPool, newP)
  const liquidity = newLiquidity - oldLiquidity

  return { newPool, liquidity, newP }
}

export function addCpmmLiquidityFixedP(
  pool: { YES: number; NO: number },
  amount: number
) {
  const prob = getCpmmProbability(pool, 0.5)
  const newPool = { ...pool }
  const sharesThrownAway = { YES: 0, NO: 0 }

  // Throws away some shares so that prob is maintained.
  if (prob < 0.5) {
    newPool.YES += amount
    newPool.NO += (prob / (1 - prob)) * amount
    sharesThrownAway.NO = amount - (prob / (1 - prob)) * amount
  } else {
    newPool.NO += amount
    newPool.YES += ((1 - prob) / prob) * amount
    sharesThrownAway.YES = amount - ((1 - prob) / prob) * amount
  }

  const oldLiquidity = getMultiCpmmLiquidity(pool)
  const newLiquidity = getMultiCpmmLiquidity(newPool)
  const liquidity = newLiquidity - oldLiquidity

  return { newPool, liquidity, sharesThrownAway }
}

export function addCpmmMultiLiquidityToAnswersIndependently(
  pools: { [answerId: string]: { YES: number; NO: number } },
  amount: number
) {
  const amountPerAnswer = amount / Object.keys(pools).length
  return mapValues(
    pools,
    (pool) => addCpmmLiquidityFixedP(pool, amountPerAnswer).newPool
  )
}

export function addCpmmMultiLiquidityAnswersSumToOne(
  pools: { [answerId: string]: { YES: number; NO: number } },
  amount: number
) {
  const answerIds = Object.keys(pools)
  const numAnswers = answerIds.length

  const newPools = { ...pools }

  let amountRemaining = amount
  while (amountRemaining > EPSILON) {
    const yesSharesThrownAway: { [answerId: string]: number } =
      Object.fromEntries(answerIds.map((answerId) => [answerId, 0]))

    for (const [answerId, pool] of Object.entries(newPools)) {
      const { newPool, sharesThrownAway } = addCpmmLiquidityFixedP(
        pool,
        amountRemaining / numAnswers
      )
      newPools[answerId] = newPool

      yesSharesThrownAway[answerId] += sharesThrownAway.YES
      const otherAnswerIds = answerIds.filter((id) => id !== answerId)
      for (const otherAnswerId of otherAnswerIds) {
        // Convert NO shares into YES shares for each other answer.
        yesSharesThrownAway[otherAnswerId] += sharesThrownAway.NO
      }
    }

    const minSharesThrownAway = Math.min(...Object.values(yesSharesThrownAway))
    amountRemaining = minSharesThrownAway
  }
  return newPools
}

export function getCpmmLiquidityPoolWeights(liquidities: LiquidityProvision[]) {
  const userAmounts = groupBy(liquidities, (w) => w.userId)
  const totalAmount = sumBy(liquidities, (w) => w.amount)

  return mapValues(
    userAmounts,
    (amounts) => sumBy(amounts, (w) => w.amount) / totalAmount
  )
}

const getK = (pool: { [outcome: string]: number }) => {
  const values = Object.values(pool)
  return sumBy(values, Math.log)
}

export const getLiquidity = (pool: { [outcome: string]: number }) => {
  return Math.exp(getK(pool) / Object.keys(pool).length)
}

export function getUserLiquidityShares(
  userId: string,
  pool: { [outcome: string]: number },
  liquidities: LiquidityProvision[]
) {
  const weights = getCpmmLiquidityPoolWeights(liquidities)
  const userWeight = weights[userId] ?? 0

  return mapValues(pool, (shares) => userWeight * shares)
}```

```
export type LiquidityProvision = {
  id: string
  userId: string
  contractId: string
  createdTime: number
  isAnte?: boolean
  // WARNING: answerId is not properly set on most LP's. It is not set on initial MC LP's even if the
  // contract has multiple answers. Furthermore, it's only set on answers added after the question was created
  // (after this commit), and house subsidies.
  answerId?: string
  amount: number // Ṁ quantity

  /** @deprecated change in constant k after provision*/
  liquidity?: number

  // For cpmm-1:
  pool?: { [outcome: string]: number } // pool shares before provision
}```

## new-bet.ts

```
import { sortBy, sumBy } from 'lodash'

import { Bet, fill, LimitBet } from './bet'
import {
  calculateCpmmAmountToProb,
  calculateCpmmAmountToProbIncludingFees,
  calculateCpmmPurchase,
  CpmmState,
  getCpmmProbability,
} from './calculate-cpmm'
import {
  BinaryContract,
  CPMMMultiContract,
  CPMMNumericContract,
  MAX_CPMM_PROB,
  MAX_STONK_PROB,
  MIN_CPMM_PROB,
  MIN_STONK_PROB,
  PseudoNumericContract,
  StonkContract,
} from './contract'
import { getFeesSplit, getTakerFee, noFees } from './fees'
import { addObjects, removeUndefinedProps } from './util/object'
import {
  floatingEqual,
  floatingGreaterEqual,
  floatingLesserEqual,
} from './util/math'
import { Answer } from './answer'
import {
  ArbitrageBetArray,
  buyNoSharesUntilAnswersSumToOne,
  calculateCpmmMultiArbitrageBet,
  calculateCpmmMultiArbitrageYesBets,
} from './calculate-cpmm-arbitrage'
import { APIError } from 'common/api/utils'

export type CandidateBet<T extends Bet = Bet> = Omit<T, 'id' | 'userId'>

export type BetInfo = {
  newBet: CandidateBet
  newPool?: { [outcome: string]: number }
  newTotalLiquidity?: number
  newP?: number
}

const computeFill = (
  amount: number,
  outcome: 'YES' | 'NO',
  limitProb: number | undefined,
  cpmmState: CpmmState,
  matchedBet: LimitBet | undefined,
  matchedBetUserBalance: number | undefined,
  freeFees?: boolean
) => {
  const prob = getCpmmProbability(cpmmState.pool, cpmmState.p)

  if (
    limitProb !== undefined &&
    (outcome === 'YES'
      ? floatingGreaterEqual(prob, limitProb) &&
        (matchedBet?.limitProb ?? 1) > limitProb
      : floatingLesserEqual(prob, limitProb) &&
        (matchedBet?.limitProb ?? 0) < limitProb)
  ) {
    // No fill.
    return undefined
  }

  const timestamp = Date.now()

  if (
    !matchedBet ||
    (outcome === 'YES'
      ? !floatingGreaterEqual(prob, matchedBet.limitProb)
      : !floatingLesserEqual(prob, matchedBet.limitProb))
  ) {
    // Fill from pool.
    const limit = !matchedBet
      ? limitProb
      : outcome === 'YES'
      ? Math.min(matchedBet.limitProb, limitProb ?? 1)
      : Math.max(matchedBet.limitProb, limitProb ?? 0)

    const buyAmount =
      limit === undefined
        ? amount
        : Math.min(
            amount,
            freeFees
              ? calculateCpmmAmountToProb(cpmmState, limit, outcome)
              : calculateCpmmAmountToProbIncludingFees(
                  cpmmState,
                  limit,
                  outcome
                )
          )

    const { shares, newPool, newP, fees } = calculateCpmmPurchase(
      cpmmState,
      buyAmount,
      outcome,
      freeFees
    )
    const newState = {
      pool: newPool,
      p: newP,
      collectedFees: addObjects(fees, cpmmState.collectedFees),
    }

    return {
      maker: {
        matchedBetId: null,
        shares,
        amount: buyAmount,
        state: newState,
        timestamp,
      },
      taker: {
        matchedBetId: null,
        shares,
        amount: buyAmount,
        timestamp,
        fees,
      },
    }
  }

  // Fill from matchedBet.
  const amountRemaining = matchedBet.orderAmount - matchedBet.amount
  const matchableUserBalance =
    matchedBetUserBalance && matchedBetUserBalance < 0
      ? 0
      : matchedBetUserBalance
  const amountToFill = Math.min(
    amountRemaining,
    matchableUserBalance ?? amountRemaining
  )

  const takerPrice =
    outcome === 'YES' ? matchedBet.limitProb : 1 - matchedBet.limitProb
  const makerPrice =
    outcome === 'YES' ? 1 - matchedBet.limitProb : matchedBet.limitProb

  const feesOnOneShare = freeFees ? 0 : getTakerFee(1, takerPrice)
  const maxTakerShares = amount / (takerPrice + feesOnOneShare)
  const maxMakerShares = amountToFill / makerPrice
  const shares = Math.min(maxTakerShares, maxMakerShares)

  const takerFee = freeFees ? 0 : getTakerFee(shares, takerPrice)
  const fees = getFeesSplit(takerFee, cpmmState.collectedFees)

  const maker = {
    bet: matchedBet,
    matchedBetId: 'taker',
    amount: shares * makerPrice,
    shares,
    timestamp,
  }
  const taker = {
    matchedBetId: matchedBet.id,
    amount: shares * takerPrice + takerFee,
    shares,
    timestamp,
    fees,
  }
  return { maker, taker }
}

export const computeFills = (
  state: CpmmState,
  outcome: 'YES' | 'NO',
  betAmount: number,
  initialLimitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number | undefined },
  limitProbs?: { max: number; min: number },
  freeFees?: boolean
) => {
  if (isNaN(betAmount)) {
    throw new Error('Invalid bet amount: ${betAmount}')
  }
  if (isNaN(initialLimitProb ?? 0)) {
    throw new Error('Invalid limitProb: ${limitProb}')
  }
  const now = Date.now()
  const { max, min } = limitProbs ?? {}
  const limit = initialLimitProb ?? (outcome === 'YES' ? max : min)
  const limitProb = !limit
    ? undefined
    : limit > MAX_CPMM_PROB
    ? MAX_CPMM_PROB
    : limit < MIN_CPMM_PROB
    ? MIN_CPMM_PROB
    : limit

  const sortedBets = sortBy(
    unfilledBets.filter(
      (bet) =>
        bet.outcome !== outcome && (bet.expiresAt ? bet.expiresAt > now : true)
    ),
    (bet) => (outcome === 'YES' ? bet.limitProb : -bet.limitProb),
    (bet) => bet.createdTime
  )

  const takers: fill[] = []
  const makers: {
    bet: LimitBet
    amount: number
    shares: number
    timestamp: number
  }[] = []
  const ordersToCancel: LimitBet[] = []

  let amount = betAmount
  let cpmmState = { ...state }
  let totalFees = noFees
  const currentBalanceByUserId = { ...balanceByUserId }

  let i = 0
  while (true) {
    const matchedBet: LimitBet | undefined = sortedBets[i]
    const fill = computeFill(
      amount,
      outcome,
      limitProb,
      cpmmState,
      matchedBet,
      currentBalanceByUserId[matchedBet?.userId ?? ''],
      freeFees
    )

    if (!fill) break

    const { taker, maker } = fill

    if (maker.matchedBetId === null) {
      // Matched against pool.
      cpmmState = maker.state
      takers.push(taker)
    } else {
      // Matched against bet.
      i++
      const { userId } = maker.bet
      const makerBalance = currentBalanceByUserId[userId]
      if (makerBalance !== undefined) {
        if (maker.amount > 0) {
          currentBalanceByUserId[userId] = makerBalance - maker.amount
        }
        const adjustedMakerBalance = currentBalanceByUserId[userId]
        if (adjustedMakerBalance !== undefined && adjustedMakerBalance <= 0) {
          // Now they've insufficient balance. Cancel maker bet.
          ordersToCancel.push(maker.bet)
        }
      }
      if (floatingEqual(maker.amount, 0)) continue

      takers.push(taker)
      makers.push(maker)
    }

    totalFees = addObjects(totalFees, taker.fees)
    amount -= taker.amount

    if (floatingEqual(amount, 0)) break
  }

  return { takers, makers, totalFees, cpmmState, ordersToCancel }
}

export const computeCpmmBet = (
  cpmmState: CpmmState,
  outcome: 'YES' | 'NO',
  initialBetAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  limitProbs?: { max: number; min: number }
) => {
  const {
    takers,
    makers,
    cpmmState: afterCpmmState,
    ordersToCancel,
    totalFees,
  } = computeFills(
    cpmmState,
    outcome,
    initialBetAmount,
    limitProb,
    unfilledBets,
    balanceByUserId,
    limitProbs
  )
  const probBefore = getCpmmProbability(cpmmState.pool, cpmmState.p)
  const probAfter = getCpmmProbability(afterCpmmState.pool, afterCpmmState.p)

  const takerAmount = sumBy(takers, 'amount')
  const takerShares = sumBy(takers, 'shares')
  const betAmount = limitProb ? initialBetAmount : takerAmount
  const isFilled = floatingEqual(betAmount, takerAmount)

  return {
    orderAmount: betAmount,
    amount: takerAmount,
    shares: takerShares,
    isFilled,
    fills: takers,
    probBefore,
    probAfter,
    makers,
    ordersToCancel,
    fees: totalFees,
    pool: afterCpmmState.pool,
    p: afterCpmmState.p,
  }
}

export const getBinaryCpmmBetInfo = (
  contract: BinaryContract | PseudoNumericContract | StonkContract,
  outcome: 'YES' | 'NO',
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  expiresAt?: number
) => {
  const cpmmState = {
    pool: contract.pool,
    p: contract.p,
    collectedFees: contract.collectedFees,
  }
  const {
    orderAmount,
    amount,
    shares,
    isFilled,
    fills,
    probBefore,
    probAfter,
    makers,
    ordersToCancel,
    pool,
    p,
    fees,
  } = computeCpmmBet(
    cpmmState,
    outcome,
    betAmount,
    limitProb,
    unfilledBets,
    balanceByUserId,
    contract.outcomeType === 'STONK'
      ? { max: MAX_STONK_PROB, min: MIN_STONK_PROB }
      : { max: MAX_CPMM_PROB, min: MIN_CPMM_PROB }
  )
  const newBet: CandidateBet = removeUndefinedProps({
    orderAmount,
    amount,
    shares,
    limitProb,
    isFilled,
    isCancelled: false,
    fills,
    contractId: contract.id,
    outcome,
    probBefore,
    probAfter,
    loanAmount: 0,
    createdTime: Date.now(),
    fees,
    isRedemption: false,
    visibility: contract.visibility,
    expiresAt,
  })

  return {
    newBet,
    newPool: pool,
    newP: p,
    makers,
    ordersToCancel,
  }
}

export const getNewMultiCpmmBetInfo = (
  contract: CPMMMultiContract | CPMMNumericContract,
  answers: Answer[],
  answer: Answer,
  outcome: 'YES' | 'NO',
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  expiresAt?: number
) => {
  if (contract.shouldAnswersSumToOne) {
    return getNewMultiCpmmBetsInfoSumsToOne(
      contract,
      answers,
      [answer],
      outcome,
      betAmount,
      limitProb,
      unfilledBets,
      balanceByUserId,
      expiresAt
    )[0]
  }

  const { poolYes, poolNo } = answer
  const pool = { YES: poolYes, NO: poolNo }
  const cpmmState = { pool, p: 0.5, collectedFees: contract.collectedFees }

  const answerUnfilledBets = unfilledBets.filter(
    (b) => b.answerId === answer.id
  )

  const {
    amount,
    fills,
    isFilled,
    makers,
    ordersToCancel,
    probAfter,
    probBefore,
    shares,
    pool: newPool,
    fees,
  } = computeCpmmBet(
    cpmmState,
    outcome,
    betAmount,
    limitProb,
    answerUnfilledBets,
    balanceByUserId,
    { max: MAX_CPMM_PROB, min: MIN_CPMM_PROB }
  )

  const newBet: CandidateBet = removeUndefinedProps({
    contractId: contract.id,
    outcome,
    orderAmount: betAmount,
    limitProb,
    isCancelled: false,
    amount,
    loanAmount: 0,
    shares,
    answerId: answer.id,
    fills,
    isFilled,
    probBefore,
    probAfter,
    createdTime: Date.now(),
    fees,
    isRedemption: false,
    visibility: contract.visibility,
    expiresAt,
  })

  return { newBet, newPool, makers, ordersToCancel }
}

export const getNewMultiCpmmBetsInfo = (
  contract: CPMMMultiContract | CPMMNumericContract,
  answers: Answer[],
  answersToBuy: Answer[],
  outcome: 'YES',
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  expiresAt?: number
) => {
  if (contract.shouldAnswersSumToOne) {
    return getNewMultiCpmmBetsInfoSumsToOne(
      contract,
      answers,
      answersToBuy,
      outcome,
      betAmount,
      limitProb,
      unfilledBets,
      balanceByUserId,
      expiresAt
    )
  } else {
    throw new APIError(400, 'Not yet implemented')
  }
}

const getNewMultiCpmmBetsInfoSumsToOne = (
  contract: CPMMMultiContract | CPMMNumericContract,
  answers: Answer[],
  answersToBuy: Answer[],
  outcome: 'YES' | 'NO',
  initialBetAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  expiresAt?: number
) => {
  const newBetResults: ArbitrageBetArray = []
  const isMultiBuy = answersToBuy.length > 1
  const otherBetsResults: ArbitrageBetArray = []
  if (answersToBuy.length === 1) {
    const { newBetResult, otherBetResults } = calculateCpmmMultiArbitrageBet(
      answers,
      answersToBuy[0],
      outcome,
      initialBetAmount,
      limitProb,
      unfilledBets,
      balanceByUserId,
      contract.collectedFees
    )
    if (newBetResult.takers.length === 0 && !limitProb) {
      throw new APIError(400, 'Betting allowed only between 1-99%.')
    }
    newBetResults.push(...([newBetResult] as ArbitrageBetArray))
    if (otherBetResults.length > 0)
      otherBetsResults.push(...(otherBetResults as ArbitrageBetArray))
  } else {
    // NOTE: only accepts YES bets atm
    const multiRes = calculateCpmmMultiArbitrageYesBets(
      answers,
      answersToBuy,
      initialBetAmount,
      limitProb,
      unfilledBets,
      balanceByUserId,
      contract.collectedFees
    )
    newBetResults.push(...multiRes.newBetResults)
    otherBetsResults.push(...multiRes.otherBetResults)
  }
  const now = Date.now()
  return newBetResults.map((newBetResult, i) => {
    const { takers, cpmmState, answer: updatedAnswer, totalFees } = newBetResult
    const probAfter = getCpmmProbability(cpmmState.pool, cpmmState.p)
    const takerAmount = sumBy(takers, 'amount')
    const takerShares = sumBy(takers, 'shares')
    const answer = answers.find((a) => a.id === updatedAnswer.id)!
    const multiBuyTakerAmount = sumBy(
      newBetResults.flatMap((r) => r.takers),
      'amount'
    )
    const betAmount = limitProb
      ? initialBetAmount
      : isMultiBuy
      ? multiBuyTakerAmount
      : takerAmount

    const newBet: CandidateBet = removeUndefinedProps({
      orderAmount: betAmount,
      amount: takerAmount,
      shares: takerShares,
      isFilled: isMultiBuy
        ? floatingEqual(multiBuyTakerAmount, betAmount)
        : floatingEqual(takerAmount, betAmount),
      fills: takers,
      contractId: contract.id,
      outcome,
      limitProb,
      isCancelled: false,
      loanAmount: 0,
      answerId: answer.id,
      probBefore: answer.prob,
      probAfter,
      createdTime: now,
      fees: totalFees,
      isRedemption: false,
      visibility: contract.visibility,
      expiresAt,
    })

    const otherResultsWithBet = otherBetsResults.map((result) => {
      const {
        answer: updatedAnswer,
        takers,
        cpmmState,
        outcome,
        totalFees,
      } = result
      const answer = answers.find((a) => a.id === updatedAnswer.id)!
      const probBefore = answer.prob
      const probAfter = getCpmmProbability(cpmmState.pool, cpmmState.p)

      const bet: CandidateBet = removeUndefinedProps({
        contractId: contract.id,
        outcome,
        orderAmount: 0,
        isCancelled: false,
        amount: 0,
        loanAmount: 0,
        shares: 0,
        answerId: answer.id,
        fills: takers,
        isFilled: true,
        probBefore,
        probAfter,
        createdTime: now,
        fees: totalFees,
        isRedemption: true,
        visibility: contract.visibility,
      })
      return {
        ...result,
        bet,
      }
    })

    return {
      newBet,
      newPool: cpmmState.pool,
      makers: newBetResult.makers,
      ordersToCancel: newBetResult.ordersToCancel,
      otherBetResults: i === 0 ? otherResultsWithBet : [],
    }
  })
}

export const getBetDownToOneMultiBetInfo = (
  contract: CPMMMultiContract,
  answers: Answer[],
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number }
) => {
  const { noBetResults, extraMana } = buyNoSharesUntilAnswersSumToOne(
    answers,
    unfilledBets,
    balanceByUserId,
    contract.collectedFees
  )

  const now = Date.now()

  const betResults = noBetResults.map((result) => {
    const { answer, takers, cpmmState, totalFees } = result
    const probBefore = answer.prob
    const probAfter = getCpmmProbability(cpmmState.pool, cpmmState.p)

    const bet: CandidateBet = removeUndefinedProps({
      contractId: contract.id,
      outcome: 'NO',
      orderAmount: 0,
      isCancelled: false,
      amount: 0,
      loanAmount: 0,
      shares: 0,
      answerId: answer.id,
      fills: takers,
      isFilled: true,
      probBefore,
      probAfter,
      createdTime: now,
      fees: totalFees,
      isRedemption: true,
      visibility: contract.visibility,
    })
    return {
      ...result,
      bet,
    }
  })

  return {
    betResults,
    extraMana,
  }
}```

## algos.ts
```
export function binarySearch(
  min: number,
  max: number,
  comparator: (x: number) => number,
  maxIterations = 15
) {
  let mid = 0
  let i = 0
  while (true) {
    mid = min + (max - min) / 2

    // Break once we've reached max precision.
    if (mid === min || mid === max) break

    const comparison = comparator(mid)
    if (comparison === 0) break
    else if (comparison > 0) {
      max = mid
    } else {
      min = mid
    }

    i++
    if (i >= maxIterations) {
      break
    }
    if (i > 100000) {
      throw new Error(
        'Binary search exceeded max iterations' +
          JSON.stringify({ min, max, mid, i }, null, 2)
      )
    }
  }
  return mid
}```

## calculate-cpmm-arbitrage
```
import { Dictionary, first, groupBy, mapValues, sum, sumBy } from 'lodash'
import { Answer } from './answer'
import { Bet, LimitBet, maker } from './bet'
import {
  calculateAmountToBuySharesFixedP,
  getCpmmProbability,
} from './calculate-cpmm'
import { binarySearch } from './util/algos'
import { computeFills } from './new-bet'
import { floatingEqual } from './util/math'
import { Fees, getFeesSplit, getTakerFee, noFees, sumAllFees } from './fees'
import { addObjects } from './util/object'
import { MAX_CPMM_PROB, MIN_CPMM_PROB } from 'common/contract'

const DEBUG = false
export type ArbitrageBetArray = ReturnType<typeof combineBetsOnSameAnswers>
const noFillsReturn = (
  outcome: string,
  answer: Answer,
  collectedFees: Fees
) => ({
  newBetResult: {
    outcome,
    answer,
    takers: [],
    makers: [] as maker[],
    ordersToCancel: [] as LimitBet[],
    cpmmState: {
      pool: { YES: answer.poolYes, NO: answer.poolNo },
      p: 0.5,
      collectedFees,
    },
    totalFees: { creatorFee: 0, liquidityFee: 0, platformFee: 0 },
  },
  otherBetResults: [] as ArbitrageBetArray,
})
export function calculateCpmmMultiArbitrageBet(
  answers: Answer[],
  answerToBuy: Answer,
  outcome: 'YES' | 'NO',
  betAmount: number,
  initialLimitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const limitProb =
    initialLimitProb !== undefined
      ? initialLimitProb
      : outcome === 'YES'
      ? MAX_CPMM_PROB
      : MIN_CPMM_PROB
  if (
    (answerToBuy.prob < MIN_CPMM_PROB && outcome === 'NO') ||
    (answerToBuy.prob > MAX_CPMM_PROB && outcome === 'YES') ||
    // Fixes limit order fills at current price when limitProb is set to a diff price and user has shares to redeem
    (answerToBuy.prob > limitProb && outcome === 'YES') ||
    (answerToBuy.prob < limitProb && outcome === 'NO')
  ) {
    return noFillsReturn(outcome, answerToBuy, collectedFees)
  }
  const result =
    outcome === 'YES'
      ? calculateCpmmMultiArbitrageBetYes(
          answers,
          answerToBuy,
          betAmount,
          limitProb,
          unfilledBets,
          balanceByUserId,
          collectedFees
        )
      : calculateCpmmMultiArbitrageBetNo(
          answers,
          answerToBuy,
          betAmount,
          limitProb,
          unfilledBets,
          balanceByUserId,
          collectedFees
        )
  if (floatingEqual(sumBy(result.newBetResult.takers, 'amount'), 0)) {
    // No trades matched.
    const { outcome, answer } = result.newBetResult
    return noFillsReturn(outcome, answer, collectedFees)
  }
  return result
}

export function calculateCpmmMultiArbitrageYesBets(
  answers: Answer[],
  answersToBuy: Answer[],
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const result = calculateCpmmMultiArbitrageBetsYes(
    answers,
    answersToBuy,
    betAmount,
    limitProb,
    unfilledBets,
    balanceByUserId,
    collectedFees
  )
  if (
    floatingEqual(
      sumBy(
        result.newBetResults.map((r) => r.takers),
        'amount'
      ),
      0
    )
  ) {
    // No trades matched.
    result.newBetResults.map((r) => {
      return {
        newBetResult: {
          outcome: r.outcome,
          answer: r.answer,
          takers: [],
          makers: [],
          ordersToCancel: [],
          cpmmState: {
            pool: { YES: r.answer.poolYes, NO: r.answer.poolNo },
            p: 0.5,
            collectedFees,
          },
          totalFees: noFees,
        },
        otherBetResults: [],
      }
    })
  }
  return result
}

export type PreliminaryBetResults = ReturnType<typeof computeFills> & {
  answer: Answer
}
function calculateCpmmMultiArbitrageBetsYes(
  initialAnswers: Answer[],
  initialAnswersToBuy: Answer[],
  initialBetAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)
  const noBetResults: PreliminaryBetResults[] = []
  const yesBetResults: PreliminaryBetResults[] = []

  let updatedAnswers = initialAnswers
  let amountToBet = initialBetAmount
  while (amountToBet > 0.01) {
    const answersToBuy = updatedAnswers.filter((a) =>
      initialAnswersToBuy.map((an) => an.id).includes(a.id)
    )
    // buy equal number of shares in each answer
    const yesSharePriceSum = sumBy(answersToBuy, 'prob')
    const maxYesShares = amountToBet / yesSharePriceSum
    let yesAmounts: number[] = []
    binarySearch(0, maxYesShares, (yesShares) => {
      yesAmounts = answersToBuy.map(({ id, poolYes, poolNo }) =>
        calculateAmountToBuySharesFixedP(
          { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
          yesShares,
          'YES',
          unfilledBetsByAnswer[id] ?? [],
          balanceByUserId
        )
      )

      const totalYesAmount = sum(yesAmounts)
      return totalYesAmount - amountToBet
    })

    const { noBuyResults, yesBets, newUpdatedAnswers } =
      getBetResultsAndUpdatedAnswers(
        answersToBuy,
        yesAmounts,
        updatedAnswers,
        limitProb,
        unfilledBets,
        balanceByUserId,
        collectedFees
      )
    updatedAnswers = newUpdatedAnswers

    amountToBet = noBuyResults.extraMana
    noBetResults.push(...noBuyResults.noBetResults)
    yesBetResults.push(...yesBets)
  }

  const noBetResultsOnBoughtAnswer = combineBetsOnSameAnswers(
    noBetResults,
    'NO',
    updatedAnswers.filter((r) =>
      initialAnswersToBuy.map((a) => a.id).includes(r.id)
    ),
    collectedFees
  )
  const extraFeesPerBoughtAnswer = Object.fromEntries(
    noBetResultsOnBoughtAnswer.map((r) => [r.answer.id, r.totalFees])
  )

  const newBetResults = combineBetsOnSameAnswers(
    yesBetResults,
    'YES',
    updatedAnswers.filter((a) =>
      initialAnswersToBuy.map((an) => an.id).includes(a.id)
    ),
    collectedFees,
    true,
    extraFeesPerBoughtAnswer
  )
  // TODO: after adding limit orders, we need to keep track of the possible matchedBetIds in the no redemption bets we're throwing away
  const otherBetResults = combineBetsOnSameAnswers(
    noBetResults,
    'NO',
    updatedAnswers.filter(
      (r) => !initialAnswersToBuy.map((a) => a.id).includes(r.id)
    ),
    collectedFees
  )

  return { newBetResults, otherBetResults, updatedAnswers }
}

export const getBetResultsAndUpdatedAnswers = (
  answersToBuy: Answer[],
  yesAmounts: number[],
  updatedAnswers: Answer[],
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees,
  answerIdsWithFees?: string[]
) => {
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)
  const yesBetResultsAndUpdatedAnswers = answersToBuy.map((answerToBuy, i) => {
    const pool = { YES: answerToBuy.poolYes, NO: answerToBuy.poolNo }
    const yesBetResult = {
      ...computeFills(
        { pool, p: 0.5, collectedFees },
        'YES',
        yesAmounts[i],
        limitProb,
        unfilledBetsByAnswer[answerToBuy.id] ?? [],
        balanceByUserId,
        undefined,
        answerIdsWithFees && !answerIdsWithFees?.includes(answerToBuy.id)
      ),
      answer: answerToBuy,
    }

    const { cpmmState } = yesBetResult
    const { pool: newPool, p } = cpmmState
    const { YES: poolYes, NO: poolNo } = newPool
    const prob = getCpmmProbability(newPool, p)
    const newAnswerState = {
      ...answerToBuy,
      poolYes,
      poolNo,
      prob,
    }
    return { yesBetResult, newAnswerState }
  })
  const yesBets = yesBetResultsAndUpdatedAnswers.map((r) => r.yesBetResult)
  const newAnswerStates = yesBetResultsAndUpdatedAnswers.map(
    (r) => r.newAnswerState
  )
  const noBuyResults = buyNoSharesUntilAnswersSumToOne(
    updatedAnswers.map(
      (answer) =>
        newAnswerStates.find(
          (newAnswerState) => newAnswerState.id === answer.id
        ) ?? answer
    ),
    unfilledBets,
    balanceByUserId,
    collectedFees,
    answerIdsWithFees
  )
  // Update new answer states from bets placed on all answers
  const newUpdatedAnswers = noBuyResults.noBetResults.map((noBetResult) => {
    const { cpmmState } = noBetResult
    const { pool: newPool, p } = cpmmState
    const { YES: poolYes, NO: poolNo } = newPool
    const prob = getCpmmProbability(newPool, p)
    return {
      ...noBetResult.answer,
      poolYes,
      poolNo,
      prob,
    }
  })

  return {
    newUpdatedAnswers,
    yesBets,
    noBuyResults,
  }
}

export const combineBetsOnSameAnswers = (
  bets: PreliminaryBetResults[],
  outcome: 'YES' | 'NO',
  updatedAnswers: Answer[],
  collectedFees: Fees,
  // The fills after the first are free bc they're due to arbitrage.
  fillsFollowingFirstAreFree?: boolean,
  extraFeesPerAnswer?: { [answerId: string]: Fees }
) => {
  return updatedAnswers.map((answer) => {
    const betsForAnswer = bets.filter((bet) => bet.answer.id === answer.id)
    const { poolYes, poolNo } = answer
    const bet = betsForAnswer[0]
    const extraFees = extraFeesPerAnswer?.[answer.id] ?? noFees
    const totalFees = betsForAnswer.reduce(
      (acc, b) => addObjects(acc, b.totalFees),
      extraFees
    )
    return {
      ...bet,
      takers: fillsFollowingFirstAreFree
        ? [
            {
              ...bet.takers[0],
              shares: sumBy(
                betsForAnswer.flatMap((r) => r.takers),
                'shares'
              ),
            },
          ]
        : betsForAnswer.flatMap((r) => r.takers),
      makers: betsForAnswer.flatMap((r) => r.makers),
      ordersToCancel: betsForAnswer.flatMap((r) => r.ordersToCancel),
      outcome,
      cpmmState: { p: 0.5, pool: { YES: poolYes, NO: poolNo }, collectedFees },
      answer,
      totalFees,
    }
  })
}

function calculateCpmmMultiArbitrageBetYes(
  answers: Answer[],
  answerToBuy: Answer,
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const startTime = Date.now()
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)

  const noSharePriceSum = sumBy(
    answers.filter((a) => a.id !== answerToBuy.id).map((a) => 1 - a.prob)
  )
  // If you spend all of amount on NO shares at current price. Subtract out from the price the redemption mana.
  const maxNoShares = betAmount / (noSharePriceSum - answers.length + 2)

  const noShares = binarySearch(0, maxNoShares, (noShares) => {
    const result = buyNoSharesInOtherAnswersThenYesInAnswer(
      answers,
      answerToBuy,
      unfilledBetsByAnswer,
      balanceByUserId,
      betAmount,
      limitProb,
      noShares,
      collectedFees
    )
    if (!result) {
      return 1
    }
    const newPools = [
      ...result.noBetResults.map((r) => r.cpmmState.pool),
      result.yesBetResult.cpmmState.pool,
    ]
    const diff = 1 - sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5))
    return diff
  })

  const result = buyNoSharesInOtherAnswersThenYesInAnswer(
    answers,
    answerToBuy,
    unfilledBetsByAnswer,
    balanceByUserId,
    betAmount,
    limitProb,
    noShares,
    collectedFees
  )
  if (!result) {
    console.log('no result', result)
    throw new Error('Invariant failed in calculateCpmmMultiArbitrageBetYes')
  }

  const { noBetResults, yesBetResult } = result

  if (DEBUG) {
    const endTime = Date.now()

    const newPools = [
      ...noBetResults.map((r) => r.cpmmState.pool),
      yesBetResult.cpmmState.pool,
    ]

    console.log('time', endTime - startTime, 'ms')

    console.log(
      'bet amount',
      betAmount,
      'no bet amounts',
      noBetResults.map((r) => r.takers.map((t) => t.amount)),
      'yes bet amount',
      sumBy(yesBetResult.takers, 'amount')
    )

    console.log(
      'getBinaryBuyYes before',
      answers.map((a) => a.prob),
      answers.map((a) => `${a.poolYes}, ${a.poolNo}`),
      'answerToBuy',
      answerToBuy
    )
    console.log(
      'getBinaryBuyYes after',
      newPools,
      newPools.map((pool) => getCpmmProbability(pool, 0.5)),
      'prob total',
      sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)),
      'pool shares',
      newPools.map((pool) => `${pool.YES}, ${pool.NO}`),
      'no shares',
      noShares,
      'yes shares',
      sumBy(yesBetResult.takers, 'shares')
    )
  }

  const newBetResult = { ...yesBetResult, outcome: 'YES' }
  const otherBetResults = noBetResults.map((r) => ({ ...r, outcome: 'NO' }))
  return { newBetResult, otherBetResults }
}

const buyNoSharesInOtherAnswersThenYesInAnswer = (
  answers: Answer[],
  answerToBuy: Answer,
  unfilledBetsByAnswer: Dictionary<LimitBet[]>,
  balanceByUserId: { [userId: string]: number },
  betAmount: number,
  limitProb: number | undefined,
  noShares: number,
  collectedFees: Fees
) => {
  const otherAnswers = answers.filter((a) => a.id !== answerToBuy.id)
  const noAmounts = otherAnswers.map(({ id, poolYes, poolNo }) =>
    calculateAmountToBuySharesFixedP(
      { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
      noShares,
      'NO',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId,
      true
    )
  )
  const totalNoAmount = sum(noAmounts)

  const noBetResults = noAmounts.map((noAmount, i) => {
    const answer = otherAnswers[i]
    const pool = { YES: answer.poolYes, NO: answer.poolNo }
    return {
      ...computeFills(
        { pool, p: 0.5, collectedFees },
        'NO',
        noAmount,
        undefined,
        unfilledBetsByAnswer[answer.id] ?? [],
        balanceByUserId,
        undefined,
        true
      ),
      answer,
    }
  })

  // Identity: No shares in all other answers is equal to noShares * (n-2) mana + yes shares in answerToBuy (quantity: noShares)
  const redeemedAmount = noShares * (answers.length - 2)
  const netNoAmount = totalNoAmount - redeemedAmount
  let yesBetAmount = betAmount - netNoAmount
  if (floatingArbitrageEqual(yesBetAmount, 0)) {
    yesBetAmount = 0
  }
  if (yesBetAmount < 0) {
    return undefined
  }

  for (const noBetResult of noBetResults) {
    const redemptionFill = {
      matchedBetId: null,
      amount: -sumBy(noBetResult.takers, 'amount'),
      shares: -sumBy(noBetResult.takers, 'shares'),
      timestamp: Date.now(),
      fees: noFees,
    }
    noBetResult.takers.push(redemptionFill)
  }

  const pool = { YES: answerToBuy.poolYes, NO: answerToBuy.poolNo }
  const yesBetResult = {
    ...computeFills(
      { pool, p: 0.5, collectedFees },
      'YES',
      yesBetAmount,
      limitProb,
      unfilledBetsByAnswer[answerToBuy.id] ?? [],
      balanceByUserId
    ),
    answer: answerToBuy,
  }

  // Redeem NO shares in other answers to YES shares in this answer.
  const redemptionFill = {
    matchedBetId: null,
    amount: netNoAmount,
    shares: noShares,
    timestamp: Date.now(),
    fees: noFees,
  }
  yesBetResult.takers.push(redemptionFill)

  return { noBetResults, yesBetResult }
}

function calculateCpmmMultiArbitrageBetNo(
  answers: Answer[],
  answerToBuy: Answer,
  betAmount: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const startTime = Date.now()
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)

  const yesSharePriceSum = sumBy(
    answers.filter((a) => a.id !== answerToBuy.id),
    'prob'
  )
  const maxYesShares = betAmount / yesSharePriceSum

  const yesShares = binarySearch(0, maxYesShares, (yesShares) => {
    const result = buyYesSharesInOtherAnswersThenNoInAnswer(
      answers,
      answerToBuy,
      unfilledBetsByAnswer,
      balanceByUserId,
      betAmount,
      limitProb,
      yesShares,
      collectedFees
    )
    if (!result) return 1
    const { yesBetResults, noBetResult } = result
    const newPools = [
      ...yesBetResults.map((r) => r.cpmmState.pool),
      noBetResult.cpmmState.pool,
    ]
    const diff = sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)) - 1
    return diff
  })

  const result = buyYesSharesInOtherAnswersThenNoInAnswer(
    answers,
    answerToBuy,
    unfilledBetsByAnswer,
    balanceByUserId,
    betAmount,
    limitProb,
    yesShares,
    collectedFees
  )
  if (!result) {
    throw new Error('Invariant failed in calculateCpmmMultiArbitrageBetNo')
  }
  const { yesBetResults, noBetResult } = result

  if (DEBUG) {
    const endTime = Date.now()

    const newPools = [
      ...yesBetResults.map((r) => r.cpmmState.pool),
      noBetResult.cpmmState.pool,
    ]

    console.log('time', endTime - startTime, 'ms')

    console.log(
      'bet amount',
      betAmount,
      'yes bet amounts',
      yesBetResults.map((r) => r.takers.map((t) => t.amount)),
      'no bet amount',
      sumBy(noBetResult.takers, 'amount')
    )

    console.log(
      'getBinaryBuyYes before',
      answers.map((a) => a.prob),
      answers.map((a) => `${a.poolYes}, ${a.poolNo}`),
      'answerToBuy',
      answerToBuy
    )
    console.log(
      'getBinaryBuyNo after',
      newPools,
      newPools.map((pool) => getCpmmProbability(pool, 0.5)),
      'prob total',
      sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)),
      'pool shares',
      newPools.map((pool) => `${pool.YES}, ${pool.NO}`),
      'yes shares',
      yesShares,
      'no shares',
      sumBy(noBetResult.takers, 'shares')
    )
  }

  const newBetResult = { ...noBetResult, outcome: 'NO' }
  const otherBetResults = yesBetResults.map((r) => ({ ...r, outcome: 'YES' }))
  return { newBetResult, otherBetResults }
}

const buyYesSharesInOtherAnswersThenNoInAnswer = (
  answers: Answer[],
  answerToBuy: Answer,
  unfilledBetsByAnswer: Dictionary<LimitBet[]>,
  balanceByUserId: { [userId: string]: number },
  betAmount: number,
  limitProb: number | undefined,
  yesShares: number,
  collectedFees: Fees
) => {
  const otherAnswers = answers.filter((a) => a.id !== answerToBuy.id)
  const yesAmounts = otherAnswers.map(({ id, poolYes, poolNo }) =>
    calculateAmountToBuySharesFixedP(
      { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
      yesShares,
      'YES',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId,
      true
    )
  )
  const totalYesAmount = sum(yesAmounts)

  const yesBetResults = yesAmounts.map((yesAmount, i) => {
    const answer = otherAnswers[i]
    const { poolYes, poolNo } = answer
    return {
      ...computeFills(
        { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
        'YES',
        yesAmount,
        undefined,
        unfilledBetsByAnswer[answer.id] ?? [],
        balanceByUserId,
        undefined,
        true
      ),
      answer,
    }
  })

  let noBetAmount = betAmount - totalYesAmount
  if (floatingArbitrageEqual(noBetAmount, 0)) {
    noBetAmount = 0
  }
  if (noBetAmount < 0) {
    return undefined
  }

  for (const yesBetResult of yesBetResults) {
    const redemptionFill = {
      matchedBetId: null,
      amount: -sumBy(yesBetResult.takers, 'amount'),
      shares: -sumBy(yesBetResult.takers, 'shares'),
      timestamp: Date.now(),
      fees: noFees,
    }
    yesBetResult.takers.push(redemptionFill)
  }

  const pool = { YES: answerToBuy.poolYes, NO: answerToBuy.poolNo }
  const noBetResult = {
    ...computeFills(
      { pool, p: 0.5, collectedFees },
      'NO',
      noBetAmount,
      limitProb,
      unfilledBetsByAnswer[answerToBuy.id] ?? [],
      balanceByUserId
    ),
    answer: answerToBuy,
  }
  // Redeem YES shares in other answers to NO shares in this answer.
  const redemptionFill = {
    matchedBetId: null,
    amount: totalYesAmount,
    shares: yesShares,
    timestamp: Date.now(),
    fees: noFees,
  }
  noBetResult.takers.push(redemptionFill)

  return { yesBetResults, noBetResult }
}

export const buyNoSharesUntilAnswersSumToOne = (
  answers: Answer[],
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees,
  answerIdsWithFees?: string[]
) => {
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)

  let maxNoShares = 10
  do {
    const result = buyNoSharesInAnswers(
      answers,
      unfilledBetsByAnswer,
      balanceByUserId,
      maxNoShares,
      collectedFees,
      answerIdsWithFees
    )
    const newPools = result.noBetResults.map((r) => r.cpmmState.pool)
    const probSum = sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5))
    if (probSum < 1) break
    maxNoShares *= 10
  } while (true)

  const noShares = binarySearch(0, maxNoShares, (noShares) => {
    const result = buyNoSharesInAnswers(
      answers,
      unfilledBetsByAnswer,
      balanceByUserId,
      noShares,
      collectedFees,
      answerIdsWithFees
    )
    const newPools = result.noBetResults.map((r) => r.cpmmState.pool)
    const diff = 1 - sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5))
    return diff
  })

  return buyNoSharesInAnswers(
    answers,
    unfilledBetsByAnswer,
    balanceByUserId,
    noShares,
    collectedFees,
    answerIdsWithFees
  )
}

const buyNoSharesInAnswers = (
  answers: Answer[],
  unfilledBetsByAnswer: Dictionary<LimitBet[]>,
  balanceByUserId: { [userId: string]: number },
  noShares: number,
  collectedFees: Fees,
  answerIdsWithFees?: string[]
) => {
  const noAmounts = answers.map(({ id, poolYes, poolNo }) =>
    calculateAmountToBuySharesFixedP(
      { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
      noShares,
      'NO',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId,
      !answerIdsWithFees?.includes(id)
    )
  )
  const totalNoAmount = sum(noAmounts)

  const noBetResults = noAmounts.map((noAmount, i) => {
    const answer = answers[i]
    const pool = { YES: answer.poolYes, NO: answer.poolNo }
    return {
      ...computeFills(
        { pool, p: 0.5, collectedFees },
        'NO',
        noAmount,
        undefined,
        unfilledBetsByAnswer[answer.id] ?? [],
        balanceByUserId,
        undefined,
        !answerIdsWithFees?.includes(answer.id)
      ),
      answer,
    }
  })
  // Identity: No shares in all other answers is equal to noShares * (n-1) mana
  const redeemedAmount = noShares * (answers.length - 1)
  // Fees on arbitrage bets are returned
  const extraMana = redeemedAmount - totalNoAmount

  for (const noBetResult of noBetResults) {
    const redemptionFill = {
      matchedBetId: null,
      amount: -sumBy(noBetResult.takers, 'amount'),
      shares: -sumBy(noBetResult.takers, 'shares'),
      timestamp: Date.now(),
      fees: noBetResult.totalFees,
    }
    noBetResult.takers.push(redemptionFill)
  }

  return { noBetResults, extraMana }
}

export function calculateCpmmMultiArbitrageSellNo(
  answers: Answer[],
  answerToSell: Answer,
  noShares: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const startTime = Date.now()
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)

  const { id, poolYes, poolNo } = answerToSell
  const pool = { YES: poolYes, NO: poolNo }
  const answersWithoutAnswerToSell = answers.filter(
    (a) => a.id !== answerToSell.id
  )

  // Strategy: We have noShares, and need that many yes shares to complete the sell.
  // We buy some yes shares in the answer directly, and the rest is from converting No shares of all the other answers.
  // The proportion of each is dependent on what leaves the final probability sum at 1.
  // Which is what this binary search is discovering.
  const yesShares = binarySearch(0, noShares, (yesShares) => {
    const noSharesInOtherAnswers = noShares - yesShares
    const yesAmount = calculateAmountToBuySharesFixedP(
      { pool, p: 0.5, collectedFees },
      yesShares,
      'YES',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId
    )
    const noAmounts = answersWithoutAnswerToSell.map(
      ({ id, poolYes, poolNo }) =>
        calculateAmountToBuySharesFixedP(
          { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
          noSharesInOtherAnswers,
          'NO',
          unfilledBetsByAnswer[id] ?? [],
          balanceByUserId,
          true
        )
    )

    const yesResult = computeFills(
      { pool, p: 0.5, collectedFees },
      'YES',
      yesAmount,
      limitProb,
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId
    )
    const noResults = answersWithoutAnswerToSell.map((answer, i) => {
      const noAmount = noAmounts[i]
      const pool = { YES: answer.poolYes, NO: answer.poolNo }
      return {
        ...computeFills(
          { pool, p: 0.5, collectedFees },
          'NO',
          noAmount,
          undefined,
          unfilledBetsByAnswer[answer.id] ?? [],
          balanceByUserId,
          undefined,
          true
        ),
        answer,
      }
    })

    const newPools = [
      yesResult.cpmmState.pool,
      ...noResults.map((r) => r.cpmmState.pool),
    ]
    const diff = sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)) - 1
    return diff
  })

  const noSharesInOtherAnswers = noShares - yesShares
  const yesAmount = calculateAmountToBuySharesFixedP(
    { pool, p: 0.5, collectedFees },
    yesShares,
    'YES',
    unfilledBetsByAnswer[id] ?? [],
    balanceByUserId
  )
  const noAmounts = answersWithoutAnswerToSell.map(({ id, poolYes, poolNo }) =>
    calculateAmountToBuySharesFixedP(
      { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
      noSharesInOtherAnswers,
      'NO',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId,
      true
    )
  )
  const yesBetResult = computeFills(
    { pool, p: 0.5, collectedFees },
    'YES',
    yesAmount,
    limitProb,
    unfilledBetsByAnswer[id] ?? [],
    balanceByUserId
  )
  const noBetResults = answersWithoutAnswerToSell.map((answer, i) => {
    const noAmount = noAmounts[i]
    const pool = { YES: answer.poolYes, NO: answer.poolNo }
    return {
      ...computeFills(
        { pool, p: 0.5, collectedFees },
        'NO',
        noAmount,
        undefined,
        unfilledBetsByAnswer[answer.id] ?? [],
        balanceByUserId,
        undefined,
        true
      ),
      answer,
    }
  })

  const redeemedMana = noSharesInOtherAnswers * (answers.length - 2)
  const netNoAmount = sum(noAmounts) - redeemedMana

  const now = Date.now()
  for (const noBetResult of noBetResults) {
    const redemptionFill = {
      matchedBetId: null,
      amount: -sumBy(noBetResult.takers, 'amount'),
      shares: -sumBy(noBetResult.takers, 'shares'),
      timestamp: now,
      fees: noFees,
    }
    noBetResult.takers.push(redemptionFill)
  }

  const arbitrageFee =
    noSharesInOtherAnswers === 0
      ? 0
      : getTakerFee(
          noSharesInOtherAnswers,
          netNoAmount / noSharesInOtherAnswers
        )
  const arbitrageFees = getFeesSplit(arbitrageFee, noFees)
  yesBetResult.takers.push({
    matchedBetId: null,
    amount: netNoAmount + arbitrageFee,
    shares: noSharesInOtherAnswers,
    timestamp: now,
    fees: arbitrageFees,
  })
  yesBetResult.totalFees = addObjects(yesBetResult.totalFees, arbitrageFees)

  if (DEBUG) {
    const endTime = Date.now()

    const newPools = [
      ...noBetResults.map((r) => r.cpmmState.pool),
      yesBetResult.cpmmState.pool,
    ]

    console.log('time', endTime - startTime, 'ms')

    console.log(
      'no shares to sell',
      noShares,
      'no bet amounts',
      noBetResults.map((r) => r.takers.map((t) => t.amount)),
      'yes bet amount',
      sumBy(yesBetResult.takers, 'amount')
    )

    console.log(
      'getBinaryBuyYes before',
      answers.map((a) => a.prob),
      answers.map((a) => `${a.poolYes}, ${a.poolNo}`),
      'answerToBuy',
      answerToSell
    )
    console.log(
      'getBinaryBuyYes after',
      newPools,
      newPools.map((pool) => getCpmmProbability(pool, 0.5)),
      'prob total',
      sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)),
      'pool shares',
      newPools.map((pool) => `${pool.YES}, ${pool.NO}`),
      'no shares',
      noShares,
      'yes shares',
      sumBy(yesBetResult.takers, 'shares')
    )
  }

  const newBetResult = { ...yesBetResult, outcome: 'YES' }
  const otherBetResults = noBetResults.map((r) => ({ ...r, outcome: 'NO' }))
  return { newBetResult, otherBetResults }
}

export function calculateCpmmMultiArbitrageSellYes(
  answers: Answer[],
  answerToSell: Answer,
  yesShares: number,
  limitProb: number | undefined,
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) {
  const startTime = Date.now()
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)

  const { id, poolYes, poolNo } = answerToSell
  const pool = { YES: poolYes, NO: poolNo }
  const answersWithoutAnswerToSell = answers.filter(
    (a) => a.id !== answerToSell.id
  )

  const noShares = binarySearch(0, yesShares, (noShares) => {
    const yesSharesInOtherAnswers = yesShares - noShares
    const noAmount = calculateAmountToBuySharesFixedP(
      { pool, p: 0.5, collectedFees },
      noShares,
      'NO',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId
    )
    const yesAmounts = answersWithoutAnswerToSell.map(
      ({ id, poolYes, poolNo }) =>
        calculateAmountToBuySharesFixedP(
          { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
          yesSharesInOtherAnswers,
          'YES',
          unfilledBetsByAnswer[id] ?? [],
          balanceByUserId,
          true
        )
    )

    const noResult = computeFills(
      { pool, p: 0.5, collectedFees },
      'NO',
      noAmount,
      limitProb,
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId
    )
    const yesResults = answersWithoutAnswerToSell.map((answer, i) => {
      const yesAmount = yesAmounts[i]
      const pool = { YES: answer.poolYes, NO: answer.poolNo }
      return {
        ...computeFills(
          { pool, p: 0.5, collectedFees },
          'YES',
          yesAmount,
          undefined,
          unfilledBetsByAnswer[answer.id] ?? [],
          balanceByUserId,
          undefined,
          true
        ),
        answer,
      }
    })

    const newPools = [
      noResult.cpmmState.pool,
      ...yesResults.map((r) => r.cpmmState.pool),
    ]
    const diff = 1 - sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5))
    return diff
  })

  const yesSharesInOtherAnswers = yesShares - noShares
  const noAmount = calculateAmountToBuySharesFixedP(
    { pool, p: 0.5, collectedFees },
    noShares,
    'NO',
    unfilledBetsByAnswer[id] ?? [],
    balanceByUserId
  )
  const yesAmounts = answersWithoutAnswerToSell.map(({ id, poolYes, poolNo }) =>
    calculateAmountToBuySharesFixedP(
      { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
      yesSharesInOtherAnswers,
      'YES',
      unfilledBetsByAnswer[id] ?? [],
      balanceByUserId,
      true
    )
  )
  const noBetResult = computeFills(
    { pool, p: 0.5, collectedFees },
    'NO',
    noAmount,
    limitProb,
    unfilledBetsByAnswer[id] ?? [],
    balanceByUserId
  )
  const yesBetResults = answersWithoutAnswerToSell.map((answer, i) => {
    const yesAmount = yesAmounts[i]
    const pool = { YES: answer.poolYes, NO: answer.poolNo }
    return {
      ...computeFills(
        { pool, p: 0.5, collectedFees },
        'YES',
        yesAmount,
        undefined,
        unfilledBetsByAnswer[answer.id] ?? [],
        balanceByUserId,
        undefined,
        true
      ),
      answer,
    }
  })

  const totalYesAmount = sum(yesAmounts)

  const now = Date.now()
  for (const yesBetResult of yesBetResults) {
    const redemptionFill = {
      matchedBetId: null,
      amount: -sumBy(yesBetResult.takers, 'amount'),
      shares: -sumBy(yesBetResult.takers, 'shares'),
      timestamp: now,
      fees: noFees,
    }
    yesBetResult.takers.push(redemptionFill)
  }

  const arbitrageFee =
    yesSharesInOtherAnswers === 0
      ? 0
      : getTakerFee(
          yesSharesInOtherAnswers,
          totalYesAmount / yesSharesInOtherAnswers
        )
  const arbitrageFees = getFeesSplit(arbitrageFee, noFees)
  noBetResult.takers.push({
    matchedBetId: null,
    amount: totalYesAmount + arbitrageFee,
    shares: yesSharesInOtherAnswers,
    timestamp: now,
    fees: arbitrageFees,
  })
  noBetResult.totalFees = addObjects(noBetResult.totalFees, arbitrageFees)

  if (DEBUG) {
    const endTime = Date.now()

    const newPools = [
      ...yesBetResults.map((r) => r.cpmmState.pool),
      noBetResult.cpmmState.pool,
    ]

    console.log('time', endTime - startTime, 'ms')

    console.log(
      'no shares to sell',
      noShares,
      'no bet amounts',
      yesBetResults.map((r) => r.takers.map((t) => t.amount)),
      'yes bet amount',
      sumBy(noBetResult.takers, 'amount')
    )

    console.log(
      'getBinaryBuyYes before',
      answers.map((a) => a.prob),
      answers.map((a) => `${a.poolYes}, ${a.poolNo}`),
      'answerToBuy',
      answerToSell
    )
    console.log(
      'getBinaryBuyYes after',
      newPools,
      newPools.map((pool) => getCpmmProbability(pool, 0.5)),
      'prob total',
      sumBy(newPools, (pool) => getCpmmProbability(pool, 0.5)),
      'pool shares',
      newPools.map((pool) => `${pool.YES}, ${pool.NO}`),
      'no shares',
      noShares,
      'yes shares',
      sumBy(noBetResult.takers, 'shares')
    )
  }

  const newBetResult = {
    ...noBetResult,
    outcome: 'NO',
  }
  const otherBetResults = yesBetResults.map((r) => ({ ...r, outcome: 'YES' }))
  return { newBetResult, otherBetResults }
}

export const calculateCpmmMultiArbitrageSellYesEqually = (
  initialAnswers: Answer[],
  userBetsByAnswerIdToSell: { [answerId: string]: Bet[] },
  unfilledBets: LimitBet[],
  balanceByUserId: { [userId: string]: number },
  collectedFees: Fees
) => {
  const unfilledBetsByAnswer = groupBy(unfilledBets, (bet) => bet.answerId)
  const allAnswersToSell = initialAnswers.filter(
    (a) => userBetsByAnswerIdToSell[a.id]?.length
  )
  const sharesByAnswerId = mapValues(userBetsByAnswerIdToSell, (bets) =>
    sumBy(bets, (b) => b.shares)
  )
  const minShares = Math.min(...Object.values(sharesByAnswerId))
  const saleBetResults: PreliminaryBetResults[] = []
  const oppositeBuyResults: PreliminaryBetResults[] = []
  let updatedAnswers = initialAnswers
  let sharesToSell = minShares
  while (sharesToSell > 0) {
    const answersToSellNow = allAnswersToSell.filter(
      (a) => sharesByAnswerId[a.id] >= sharesToSell
    )
    const answerIdsToSellNow = allAnswersToSell
      .filter((a) => sharesByAnswerId[a.id] >= sharesToSell)
      .map((a) => a.id)
    // Buy yes shares in the answers opposite the answers to sell
    const oppositeAnswersFromSaleToBuyYesShares = updatedAnswers.filter(
      (a) => !answerIdsToSellNow.includes(a.id)
    )
    let saleBets: PreliminaryBetResults[]
    if (answersToSellNow.length !== initialAnswers.length) {
      const yesAmounts = oppositeAnswersFromSaleToBuyYesShares.map(
        ({ id, poolYes, poolNo }) => {
          return calculateAmountToBuySharesFixedP(
            { pool: { YES: poolYes, NO: poolNo }, p: 0.5, collectedFees },
            sharesToSell,
            'YES',
            unfilledBetsByAnswer[id] ?? [],
            balanceByUserId,
            // Zero fees on arbitrage bets
            true
          )
        }
      )
      const { newUpdatedAnswers, yesBets, noBuyResults } =
        getBetResultsAndUpdatedAnswers(
          oppositeAnswersFromSaleToBuyYesShares,
          yesAmounts,
          updatedAnswers,
          undefined,
          unfilledBets,
          balanceByUserId,
          collectedFees,
          // Charge fees on sale bets
          answerIdsToSellNow
        )
      updatedAnswers = newUpdatedAnswers
      for (const yesBet of yesBets) {
        const redemptionFill = {
          matchedBetId: null,
          amount: -sumBy(yesBet.takers, 'amount'),
          shares: -sumBy(yesBet.takers, 'shares'),
          timestamp: first(yesBet.takers)?.timestamp ?? Date.now(),
          fees: yesBet.totalFees,
        }
        yesBet.takers.push(redemptionFill)
      }
      oppositeBuyResults.push(...yesBets)
      const totalYesAmount = sum(yesAmounts)
      const { noBetResults, extraMana } = noBuyResults
      saleBets = noBetResults
        // TODO: after adding limit orders, we need to keep track of the matchedBetIds in the redemption bets we're throwing away
        .filter((betResult) => answerIdsToSellNow.includes(betResult.answer.id))
        .map((betResult) => {
          const answer = updatedAnswers.find(
            (a) => a.id === betResult.answer.id
          )!
          const { poolYes, poolNo } = answer
          return {
            ...betResult,
            takers: [
              {
                matchedBetId: null,
                amount:
                  -(sharesToSell - totalYesAmount + extraMana) /
                  answerIdsToSellNow.length,
                shares: -sharesToSell,
                timestamp: first(betResult.takers)?.timestamp ?? Date.now(),
                isSale: true,
                fees: betResult.totalFees,
              },
              //...betResult.takers, these are takers in the opposite outcome, not sure where to put them
            ],
            cpmmState: {
              p: 0.5,
              pool: { YES: poolYes, NO: poolNo },
              collectedFees,
            },
            answer,
          }
        })
    } else {
      // If we have yes shares in ALL answers, redeem them for mana
      saleBets = getSellAllRedemptionPreliminaryBets(
        answersToSellNow,
        sharesToSell,
        collectedFees,
        Date.now()
      )
    }
    saleBetResults.push(...saleBets)
    for (const answerIdToSell of answerIdsToSellNow) {
      sharesByAnswerId[answerIdToSell] -= sharesToSell
    }
    const answersToSellRemaining = Object.values(sharesByAnswerId).filter(
      (shares) => shares > 0
    )
    if (answersToSellRemaining.length === 0) break
    sharesToSell = Math.min(...answersToSellRemaining)
  }

  const newBetResults = combineBetsOnSameAnswers(
    saleBetResults,
    'YES',
    updatedAnswers.filter((a) =>
      allAnswersToSell.map((an) => an.id).includes(a.id)
    ),
    collectedFees
  )

  const otherBetResults = combineBetsOnSameAnswers(
    oppositeBuyResults,
    'YES',
    updatedAnswers.filter(
      (r) => !allAnswersToSell.map((a) => a.id).includes(r.id)
    ),
    collectedFees
  )
  const totalFee = sumAllFees(
    newBetResults.concat(otherBetResults).map((r) => r.totalFees)
  )

  return { newBetResults, otherBetResults, updatedAnswers, totalFee }
}

export const getSellAllRedemptionPreliminaryBets = (
  answers: Answer[],
  sharesToSell: number,
  collectedFees: Fees,
  now: number
) => {
  return answers.map((answer) => {
    const { poolYes, poolNo } = answer
    return {
      outcome: 'YES' as const,
      takers: [
        {
          matchedBetId: null,
          amount: -sharesToSell / answers.length,
          shares: -sharesToSell,
          timestamp: now,
          isSale: true,
          fees: noFees,
        },
      ],
      makers: [],
      totalFees: noFees,
      cpmmState: { p: 0.5, pool: { YES: poolYes, NO: poolNo }, collectedFees },
      ordersToCancel: [],
      answer,
    }
  })
}

export function floatingArbitrageEqual(a: number, b: number, epsilon = 0.001) {
  return Math.abs(a - b) < epsilon
}```

## answers.ts

```
import { sortBy } from 'lodash'
import { MultiContract, resolution } from './contract'

export type Answer = {
  id: string
  index: number // Order of the answer in the list
  contractId: string
  userId: string
  text: string
  createdTime: number
  color?: string // Hex color override in UI

  // Mechanism props
  poolYes: number // YES shares
  poolNo: number // NO shares
  prob: number // Computed from poolYes and poolNo.
  totalLiquidity: number // for historical reasons, this the total subsidy amount added in M
  subsidyPool: number // current value of subsidy pool in M

  // Is this 'Other', the answer that represents all other answers, including answers added in the future.
  isOther?: boolean

  resolution?: resolution
  resolutionTime?: number
  resolutionProbability?: number
  resolverId?: string

  probChanges: {
    day: number
    week: number
    month: number
  }

  loverUserId?: string
}

export const MAX_ANSWER_LENGTH = 240

export const MAX_ANSWERS = 100
export const MAX_INDEPENDENT_ANSWERS = 200

export const getMaximumAnswers = (shouldAnswersSumToOne: boolean) =>
  shouldAnswersSumToOne ? MAX_ANSWERS : MAX_INDEPENDENT_ANSWERS

export const OTHER_TOOLTIP_TEXT =
  "Bet on all answers that aren't listed yet. A bet on Other automatically includes any answer added in the future."

export type MultiSort =
  | 'prob-desc'
  | 'prob-asc'
  | 'old'
  | 'new'
  | 'liquidity'
  | 'alphabetical'

export const getDefaultSort = (contract: MultiContract) => {
  const { sort, answers } = contract
  if (sort) return sort
  if (contract.addAnswersMode === 'DISABLED') return 'old'
  else if (!contract.shouldAnswersSumToOne) return 'prob-desc'
  else if (answers.length > 10) return 'prob-desc'
  return 'old'
}

export const sortAnswers = <T extends Answer>(
  contract: MultiContract,
  answers: T[],
  sort?: MultiSort
) => {
  const { resolutions } = contract
  sort = sort ?? getDefaultSort(contract)

  const shouldAnswersSumToOne =
    'shouldAnswersSumToOne' in contract ? contract.shouldAnswersSumToOne : true

  return sortBy(answers, [
    shouldAnswersSumToOne
      ? // Winners first
        (answer) => (resolutions ? -1 * resolutions[answer.id] : answer)
      : // Resolved last
        (answer) => (answer.resolution ? 1 : 0),
    // then by sort
    (answer) => {
      if (sort === 'old') {
        return answer.resolutionTime ? answer.resolutionTime : answer.index
      } else if (sort === 'new') {
        return answer.resolutionTime ? -answer.resolutionTime : -answer.index
      } else if (sort === 'prob-asc') {
        return answer.prob
      } else if (sort === 'prob-desc') {
        return -1 * answer.prob
      } else if (sort === 'liquidity') {
        return answer.subsidyPool ? -1 * answer.subsidyPool : 0
      } else if (sort === 'alphabetical') {
        return answer.text.toLowerCase()
      }
      return 0
    },
  ])
}
```
Please modify the earlier python code to incorporate the same logic as the code here