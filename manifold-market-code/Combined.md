## algos.ts

```ts
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
}
```

## answer.ts

```ts
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

## bet.ts

```ts
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

## calculate-cpmm-arbitrage.ts

```ts
import { Dictionary, first, groupBy, mapValues, sum, sumBy } from 'lodash'
import { Answer } from './answer'
import { Bet, LimitBet, maker } from './bet'
import {
  calculateAmountToBuySharesFixedP,
  getCpmmProbability,
} from './calculate-cpmm'
import { binarySearch } from './algos'
import { computeFills } from './new-bet'
import { floatingEqual } from './math'
import { Fees, getFeesSplit, getTakerFee, noFees, sumAllFees } from './fees'
import { addObjects } from './object'
import { MAX_CPMM_PROB, MIN_CPMM_PROB } from './contract'

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
}

```

## calculate-cpmm.ts

```ts
import { groupBy, mapValues, sumBy } from 'lodash'
import { LimitBet } from './bet'

import { Fees, getFeesSplit, getTakerFee, noFees } from './fees'
import { LiquidityProvision } from './liquidity-provision'
import { computeFills } from './new-bet'
import { binarySearch } from './algos'
import { EPSILON, floatingEqual } from './math'
import {
  calculateCpmmMultiArbitrageSellNo,
  calculateCpmmMultiArbitrageSellYes,
} from './calculate-cpmm-arbitrage'
import { Answer } from './answer'
import { CPMMContract, CPMMMultiContract } from './contract'

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
}

```

## calculate-fixed-payouts.ts

```ts
import { sum } from 'lodash'
import { Bet } from './bet'
import { getProbability } from './calculate'
import {
  CPMMContract,
  CPMMMultiContract,
  CPMMNumericContract,
} from './contract'

export function calculateFixedPayout(
  contract: CPMMContract,
  bet: Bet,
  outcome: string
) {
  if (outcome === 'CANCEL') return calculateFixedCancelPayout(bet)
  if (outcome === 'MKT') return calculateFixedMktPayout(contract, bet)

  return calculateStandardFixedPayout(bet, outcome)
}

export function calculateFixedCancelPayout(bet: Bet) {
  return bet.amount
}

export function calculateStandardFixedPayout(bet: Bet, outcome: string) {
  const { outcome: betOutcome, shares } = bet

  if (betOutcome !== outcome) return 0
  return shares
}

function calculateFixedMktPayout(contract: CPMMContract, bet: Bet) {
  const { resolutionProbability } = contract
  const prob =
    resolutionProbability !== undefined
      ? resolutionProbability
      : getProbability(contract)

  const { outcome, shares } = bet
  const betP = outcome === 'YES' ? prob : 1 - prob
  return betP * shares
}

function calculateBetPayoutMulti(
  contract: CPMMMultiContract | CPMMNumericContract,
  bet: Bet
) {
  let prob = 0
  const { answerId } = bet
  if (answerId) {
    const { answers, resolutions } = contract
    const answer = answers.find((a) => a.id === answerId)
    if (answer && answer.resolution) {
      const { resolution, resolutionProbability } = answer
      if (resolution === 'YES') prob = 1
      else if (resolution === 'NO') prob = 0
      else if (resolutionProbability) prob = resolutionProbability
    } else if (resolutions) {
      const probTotal = sum(Object.values(resolutions))
      prob = (resolutions[answerId] ?? 0) / probTotal
    } else if (answer) {
      prob = answer.prob
    }
  }
  const { outcome, shares } = bet
  const betP = outcome === 'YES' ? prob : 1 - prob
  return betP * shares
}

export function calculateFixedPayoutMulti(
  contract: CPMMMultiContract | CPMMNumericContract,
  bet: Bet,
  outcome: string
) {
  if (outcome === 'CANCEL') return calculateFixedCancelPayout(bet)
  return calculateBetPayoutMulti(contract, bet)
}

```

## calculate-metrics.ts

## calculate.ts

```ts
import {
  get,
  groupBy,
  mapValues,
  maxBy,
  partition,
  sortBy,
  sum,
  sumBy,
} from 'lodash'
import { Bet } from './bet'
import {
  calculateCpmmPurchase,
  getCpmmOutcomeProbabilityAfterBet,
  getCpmmProbability,
} from './calculate-cpmm'
import {
  calculateFixedPayout,
  calculateFixedPayoutMulti,
} from './calculate-fixed-payouts'
import {
  BinaryContract,
  Contract,
  CPMMContract,
  CPMMMultiContract,
  CPMMNumericContract,
  MultiContract,
  PseudoNumericContract,
  StonkContract,
} from './contract'
import { floatingEqual, floatingGreaterEqual } from './math'
import { ContractMetric } from './contract-metric'
import { Answer } from './answer'
import { DAY_MS } from './time'
import { computeInvestmentValueCustomProb } from './calculate-metrics'

export function getProbability(
  contract: BinaryContract | PseudoNumericContract | StonkContract
) {
  return getCpmmProbability(contract.pool, contract.p)
}

export function getDisplayProbability(
  contract: BinaryContract | PseudoNumericContract | StonkContract
) {
  return contract.resolutionProbability ?? getProbability(contract)
}

export function getInitialProbability(
  contract: BinaryContract | PseudoNumericContract | StonkContract
) {
  if (contract.initialProbability) return contract.initialProbability

  return getCpmmProbability(contract.pool, contract.p)
}

export function getOutcomeProbability(contract: Contract, outcome: string) {
  const { mechanism } = contract
  switch (mechanism) {
    case 'cpmm-1':
      return outcome === 'YES'
        ? getCpmmProbability(contract.pool, contract.p)
        : 1 - getCpmmProbability(contract.pool, contract.p)
    case 'cpmm-multi-1':
      return 0
    default:
      throw new Error('getOutcomeProbability not implemented')
  }
}

export function getAnswerProbability(
  contract: MultiContract,
  answerId: string
) {
  const answer = contract.answers.find((a) => a.id === answerId)
  if (!answer) return 0

  const { poolYes, poolNo, resolution, resolutionProbability } = answer
  if (resolution) {
    if (resolution === 'MKT') return resolutionProbability ?? answer.prob
    if (resolution === 'YES') return 1
    if (resolution === 'NO') return 0
  }
  const pool = { YES: poolYes, NO: poolNo }
  return getCpmmProbability(pool, 0.5)
}

export function getInitialAnswerProbability(
  contract: MultiContract | CPMMNumericContract,
  answer: Answer
) {
  if (!contract.shouldAnswersSumToOne) {
    return 0.5
  } else {
    if (contract.addAnswersMode === 'DISABLED') {
      return 1 / contract.answers.length
    } else {
      const answers = contract.answers
      const initialTime = answers.find((a) => a.isOther)?.createdTime

      if (answer.createdTime === initialTime) {
        const numberOfInitialAnswers = sumBy(answers, (a) =>
          a.createdTime === initialTime ? 1 : 0
        )
        return 1 / numberOfInitialAnswers
      }
      return undefined
    }
  }
}

export function getOutcomeProbabilityAfterBet(
  contract: Contract,
  outcome: string,
  bet: number
) {
  const { mechanism } = contract
  switch (mechanism) {
    case 'cpmm-1':
      return getCpmmOutcomeProbabilityAfterBet(contract, outcome, bet)
    case 'cpmm-multi-1':
      return 0
    default:
      throw new Error('getOutcomeProbabilityAfterBet not implemented')
  }
}

export function calculateSharesBought(
  contract: Contract,
  outcome: string,
  amount: number
) {
  const { mechanism } = contract
  switch (mechanism) {
    case 'cpmm-1':
      return calculateCpmmPurchase(contract, amount, outcome).shares
    default:
      throw new Error('calculateSharesBought not implemented')
  }
}

export function calculatePayout(contract: Contract, bet: Bet, outcome: string) {
  const { mechanism } = contract
  return mechanism === 'cpmm-1'
    ? calculateFixedPayout(contract, bet, outcome)
    : mechanism === 'cpmm-multi-1'
    ? calculateFixedPayoutMulti(contract, bet, outcome)
    : bet?.amount ?? 0
}

export function resolvedPayout(contract: Contract, bet: Bet) {
  const { resolution, mechanism } = contract
  if (!resolution) throw new Error('Contract not resolved')

  return mechanism === 'cpmm-1'
    ? calculateFixedPayout(contract, bet, resolution)
    : mechanism === 'cpmm-multi-1'
    ? calculateFixedPayoutMulti(contract, bet, resolution)
    : bet?.amount ?? 0
}

function getCpmmInvested(yourBets: Bet[]) {
  const totalShares: { [outcome: string]: number } = {}
  const totalSpent: { [outcome: string]: number } = {}

  const sharePurchases = sortBy(yourBets, [
    'createdTime',
    (bet) => (bet.isRedemption ? 1 : 0),
  ])

  for (const purchase of sharePurchases) {
    const { outcome, shares, amount } = purchase
    if (floatingEqual(shares, 0)) continue

    const spent = totalSpent[outcome] ?? 0
    const position = totalShares[outcome] ?? 0

    if (floatingGreaterEqual(amount, 0)) {
      totalShares[outcome] = position + shares
      totalSpent[outcome] = spent + amount
    } else {
      const averagePrice = floatingEqual(position, 0) ? 0 : spent / position
      totalShares[outcome] = position + shares
      totalSpent[outcome] = spent + averagePrice * shares
    }
  }

  return sum(Object.values(totalSpent))
}

export function getSimpleCpmmInvested(yourBets: Bet[]) {
  const total = sumBy(yourBets, (b) => b.amount)
  if (total < 0) return 0
  return total
}

export function getInvested(contract: Contract, yourBets: Bet[]) {
  const { mechanism } = contract
  if (mechanism === 'cpmm-1') return getCpmmInvested(yourBets)
  if (mechanism === 'cpmm-multi-1') {
    const betsByAnswerId = groupBy(yourBets, 'answerId')
    const investedByAnswerId = mapValues(betsByAnswerId, getCpmmInvested)
    return sum(Object.values(investedByAnswerId))
  }
  throw new Error('getInvested not implemented for mechanism ' + mechanism)
}

function getCpmmOrDpmProfit(
  contract: Contract,
  yourBets: Bet[],
  answer?: Answer
) {
  const resolution = answer?.resolution ?? contract.resolution

  let totalInvested = 0
  let payout = 0
  let saleValue = 0
  let redeemed = 0

  for (const bet of yourBets) {
    const { amount, isRedemption } = bet

    if (isRedemption) {
      redeemed += -1 * amount
    } else if (amount > 0) {
      totalInvested += amount
    } else {
      saleValue -= amount
    }

    payout += resolution
      ? calculatePayout(contract, bet, resolution)
      : calculatePayout(contract, bet, 'MKT')
  }

  const profit = payout + saleValue + redeemed - totalInvested
  const profitPercent = totalInvested === 0 ? 0 : (profit / totalInvested) * 100

  return {
    profit,
    profitPercent,
    totalInvested,
    payout,
  }
}

export function getProfitMetrics(contract: Contract, yourBets: Bet[]) {
  const { mechanism } = contract
  if (mechanism === 'cpmm-multi-1') {
    const betsByAnswerId = groupBy(yourBets, 'answerId')
    const profitMetricsPerAnswer = Object.entries(betsByAnswerId).map(
      ([answerId, bets]) => {
        const answer = contract.answers.find((a) => a.id === answerId)
        return getCpmmOrDpmProfit(contract, bets, answer)
      }
    )
    const profit = sumBy(profitMetricsPerAnswer, 'profit')
    const totalInvested = sumBy(profitMetricsPerAnswer, 'totalInvested')
    const profitPercent =
      totalInvested === 0 ? 0 : (profit / totalInvested) * 100
    const payout = sumBy(profitMetricsPerAnswer, 'payout')
    return {
      profit,
      profitPercent,
      totalInvested,
      payout,
    }
  }
  return getCpmmOrDpmProfit(contract, yourBets)
}

export function getCpmmShares(yourBets: Bet[]) {
  const totalShares: { [outcome: string]: number } = {}
  for (const bet of yourBets) {
    const { shares, outcome } = bet
    totalShares[outcome] = (totalShares[outcome] ?? 0) + shares
  }

  const hasShares = Object.values(totalShares).some(
    (shares) => !floatingEqual(shares, 0)
  )

  const { YES: yesShares, NO: noShares } = totalShares
  const hasYesShares = yesShares >= 1
  const hasNoShares = noShares >= 1

  return {
    totalShares,
    hasShares,
    hasYesShares,
    hasNoShares,
  }
}

export function getCpmmMultiShares(yourBets: Bet[]) {
  const betsByAnswerId = groupBy(yourBets, 'answerId')
  const sharesByAnswerId = mapValues(betsByAnswerId, (bets) =>
    getCpmmShares(bets)
  )

  const hasShares = Object.values(sharesByAnswerId).some(
    (shares) => shares.hasShares
  )

  return {
    hasShares,
    sharesByAnswerId,
  }
}

export const getContractBetMetrics = (
  contract: Contract,
  yourBets: Bet[],
  answerId?: string
) => {
  const { mechanism } = contract
  const isCpmmMulti = mechanism === 'cpmm-multi-1'
  const { profit, profitPercent, payout } = getProfitMetrics(contract, yourBets)
  const invested = getInvested(contract, yourBets)
  const loan = sumBy(yourBets, 'loanAmount')

  const { totalShares, hasShares, hasYesShares, hasNoShares } =
    getCpmmShares(yourBets)
  const lastBetTime = Math.max(...yourBets.map((b) => b.createdTime))
  const maxSharesOutcome = hasShares
    ? maxBy(Object.keys(totalShares), (outcome) => totalShares[outcome])
    : null

  return {
    invested,
    loan,
    payout,
    profit,
    profitPercent,
    totalShares,
    hasShares: isCpmmMulti ? getCpmmMultiShares(yourBets).hasShares : hasShares,
    hasYesShares,
    hasNoShares,
    maxSharesOutcome,
    lastBetTime,
    answerId: answerId ?? null,
  }
}
export const getContractBetMetricsPerAnswer = (
  contract: Contract,
  bets: Bet[],
  answers?: Answer[]
) => {
  const betsPerAnswer = groupBy(bets, 'answerId')
  const metricsPerAnswer = Object.values(
    mapValues(betsPerAnswer, (bets) => {
      const periods = ['day', 'week', 'month'] as const
      const answerId = bets[0].answerId
      const baseMetrics = getContractBetMetrics(contract, bets, answerId)
      let periodMetrics
      if (
        contract.mechanism === 'cpmm-1' ||
        contract.mechanism === 'cpmm-multi-1'
      ) {
        const answer = answers?.find((a) => a.id === answerId)
        const passedAnswer = !!answer
        if (contract.mechanism === 'cpmm-multi-1' && !passedAnswer) {
          console.log(
            `answer with id ${bets[0].answerId} not found, but is required for cpmm-multi-1 contract: ${contract.id}`
          )
        } else {
          periodMetrics = Object.fromEntries(
            periods.map((period) => [
              period,
              calculatePeriodProfit(contract, bets, period, answer),
            ])
          )
        }
      }
      return {
        ...baseMetrics,
        from: periodMetrics,
      } as ContractMetric
    })
  )

  // Calculate overall contract metrics with answerId:null bc it's nice to have
  if (contract.mechanism === 'cpmm-multi-1') {
    const baseFrom = metricsPerAnswer[0].from
    const calculateProfitPercent = (
      metrics: ContractMetric[],
      period: string
    ) => {
      const profit = sumBy(metrics, (m) => get(m, `from.${period}.profit`, 0))
      const invested = sumBy(metrics, (m) =>
        get(m, `from.${period}.invested`, 0)
      )
      return invested !== 0 ? 100 * (profit / invested) : 0
    }

    const baseMetric = getContractBetMetrics(contract, bets)
    const from = baseFrom
      ? mapValues(baseFrom, (periodMetrics, period) =>
          mapValues(periodMetrics, (_, key) =>
            key === 'profitPercent'
              ? calculateProfitPercent(metricsPerAnswer, period)
              : sumBy(metricsPerAnswer, (m) =>
                  get(m, `from.${period}.${key}`, 0)
                )
          )
        )
      : undefined
    metricsPerAnswer.push({
      ...baseMetric,
      // Overall period metrics = sum all the answers' period metrics
      from,
      answerId: null,
    } as ContractMetric)
  }
  return metricsPerAnswer
}

const calculatePeriodProfit = (
  contract: CPMMContract | CPMMMultiContract | CPMMNumericContract,
  bets: Bet[],
  period: 'day' | 'week' | 'month',
  answer?: Answer
) => {
  const days = period === 'day' ? 1 : period === 'week' ? 7 : 30
  const fromTime = Date.now() - days * DAY_MS
  const [previousBets, recentBets] = partition(
    bets,
    (b) => b.createdTime < fromTime
  )

  const { prob, probChanges } = answer ?? (contract as CPMMContract)
  const prevProb = prob - probChanges[period]

  const previousBetsValue = computeInvestmentValueCustomProb(
    previousBets,
    contract,
    prevProb
  )
  const currentBetsValue = computeInvestmentValueCustomProb(
    previousBets,
    contract,
    prob
  )

  const { profit: recentProfit, invested: recentInvested } =
    getContractBetMetrics(contract, recentBets)

  const profit = currentBetsValue - previousBetsValue + recentProfit
  const invested = previousBetsValue + recentInvested
  const profitPercent = invested === 0 ? 0 : 100 * (profit / invested)

  return {
    profit,
    profitPercent,
    invested,
    prevValue: previousBetsValue,
    value: currentBetsValue,
  }
}

export function getContractBetNullMetrics() {
  return {
    invested: 0,
    loan: 0,
    payout: 0,
    profit: 0,
    profitPercent: 0,
    totalShares: {} as { [outcome: string]: number },
    hasShares: false,
    hasYesShares: false,
    hasNoShares: false,
    maxSharesOutcome: null,
  } as ContractMetric
}

```

```ts
import { Dictionary, min, sumBy, uniq } from 'lodash'
import { calculatePayout, getContractBetMetricsPerAnswer } from './calculate'
import { Bet, LimitBet } from './bet'
import {
  Contract,
  CPMMMultiContract,
  CPMMMultiNumeric,
  getAdjustedProfit,
} from './contract'
import { User } from './user'
import { computeFills } from './new-bet'
import { CpmmState, getCpmmProbability } from './calculate-cpmm'
import { removeUndefinedProps } from './object'
import { logit } from './math'
import { ContractMetric } from './contract-metric'
import { Answer } from './answer'
import { noFees } from './fees'
import { DisplayUser } from './api/user-types'

export const computeInvestmentValue = (
  bets: Bet[],
  contractsDict: { [k: string]: Contract }
) => {
  let investmentValue = 0
  let cashInvestmentValue = 0
  for (const bet of bets) {
    const contract = contractsDict[bet.contractId]
    if (!contract || contract.isResolved) continue

    let payout
    try {
      payout = calculatePayout(contract, bet, 'MKT')
    } catch (e) {
      console.log(
        'contract',
        contract.question,
        contract.mechanism,
        contract.id
      )
      console.error(e)
      payout = 0
    }
    const value = payout - (bet.loanAmount ?? 0)
    if (isNaN(value)) continue

    if (contract.token === 'CASH') {
      cashInvestmentValue += value
    } else {
      investmentValue += value
    }
  }

  return { investmentValue, cashInvestmentValue }
}

export const computeInvestmentValueCustomProb = (
  bets: Bet[],
  contract: Contract,
  p: number
) => {
  return sumBy(bets, (bet) => {
    if (!contract) return 0
    const { outcome, shares } = bet

    const betP = outcome === 'YES' ? p : 1 - p

    const value = betP * shares
    if (isNaN(value)) return 0
    return value
  })
}

const getLoanTotal = (
  bets: Bet[],
  contractsDict: { [k: string]: Contract }
) => {
  return sumBy(bets, (bet) => {
    const contract = contractsDict[bet.contractId]
    if (!contract || contract.isResolved) return 0
    return bet.loanAmount ?? 0
  })
}

export const ELASTICITY_BET_AMOUNT = 10000 // readjust with platform volume

export const computeElasticity = (
  unfilledBets: LimitBet[],
  contract: Contract,
  betAmount = ELASTICITY_BET_AMOUNT
) => {
  const { mechanism, isResolved } = contract

  switch (mechanism) {
    case 'cpmm-1':
      return computeBinaryCpmmElasticity(
        isResolved ? [] : unfilledBets, // only consider limit orders for open markets
        contract,
        betAmount
      )
    case 'cpmm-multi-1':
      return computeMultiCpmmElasticity(
        isResolved ? [] : unfilledBets, // only consider limit orders for open markets
        contract,
        betAmount
      )
    default: // there are some contracts on the dev DB with crazy mechanisms
      return 1_000_000
  }
}

export const computeBinaryCpmmElasticity = (
  unfilledBets: LimitBet[],
  cpmmState: CpmmState,
  betAmount: number
) => {
  const sortedBets = unfilledBets.sort((a, b) => a.createdTime - b.createdTime)

  const userIds = uniq(unfilledBets.map((b) => b.userId))
  // Assume all limit orders are good.
  const userBalances = Object.fromEntries(
    userIds.map((id) => [id, Number.MAX_SAFE_INTEGER])
  )

  const {
    cpmmState: { pool: poolY, p: pY },
  } = computeFills(
    cpmmState,
    'YES',
    betAmount,
    undefined,
    sortedBets,
    userBalances
  )
  const resultYes = getCpmmProbability(poolY, pY)

  const {
    cpmmState: { pool: poolN, p: pN },
  } = computeFills(
    cpmmState,
    'NO',
    betAmount,
    undefined,
    sortedBets,
    userBalances
  )
  const resultNo = getCpmmProbability(poolN, pN)

  // handle AMM overflow
  const safeYes = Number.isFinite(resultYes)
    ? Math.min(resultYes, 0.995)
    : 0.995
  const safeNo = Number.isFinite(resultNo) ? Math.max(resultNo, 0.005) : 0.005

  return logit(safeYes) - logit(safeNo)
}

export const computeBinaryCpmmElasticityFromAnte = (
  ante: number,
  betAmount = ELASTICITY_BET_AMOUNT
) => {
  const pool = { YES: ante, NO: ante }
  const p = 0.5

  const cpmmState = {
    pool,
    p,
    collectedFees: noFees,
  }

  const {
    cpmmState: { pool: poolY, p: pY },
  } = computeFills(cpmmState, 'YES', betAmount, undefined, [], {})
  const resultYes = getCpmmProbability(poolY, pY)

  const {
    cpmmState: { pool: poolN, p: pN },
  } = computeFills(cpmmState, 'NO', betAmount, undefined, [], {})
  const resultNo = getCpmmProbability(poolN, pN)

  // handle AMM overflow
  const safeYes = Number.isFinite(resultYes) ? resultYes : 1
  const safeNo = Number.isFinite(resultNo) ? resultNo : 0

  return logit(safeYes) - logit(safeNo)
}

const computeMultiCpmmElasticity = (
  unfilledBets: LimitBet[],
  contract: CPMMMultiContract | CPMMMultiNumeric,
  betAmount: number
) => {
  const elasticities = contract.answers.map((a) => {
    const cpmmState = {
      pool: { YES: a.poolYes, NO: a.poolNo },
      p: 0.5,
      collectedFees: noFees,
    }
    const unfilledBetsForAnswer = unfilledBets.filter(
      (b) => b.answerId === a.id
    )
    return computeBinaryCpmmElasticity(
      unfilledBetsForAnswer,
      cpmmState,
      betAmount
    )
  })
  return min(elasticities) ?? 1_000_000
}

export const calculateNewPortfolioMetrics = (
  user: User,
  contractsById: { [k: string]: Contract },
  unresolvedBets: Bet[]
) => {
  const { investmentValue, cashInvestmentValue } = computeInvestmentValue(
    unresolvedBets,
    contractsById
  )
  const loanTotal = getLoanTotal(unresolvedBets, contractsById)
  return {
    investmentValue,
    cashInvestmentValue,
    balance: user.balance,
    cashBalance: user.cashBalance,
    spiceBalance: user.spiceBalance,
    totalDeposits: user.totalDeposits,
    totalCashDeposits: user.totalCashDeposits,
    loanTotal,
    timestamp: Date.now(),
    userId: user.id,
  }
}

export const calculateMetricsByContractAndAnswer = (
  betsByContractId: Dictionary<Bet[]>,
  contractsById: Dictionary<Contract>,
  user: User,
  answersByContractId: Dictionary<Answer[]>
) => {
  return Object.entries(betsByContractId).map(([contractId, bets]) => {
    const contract: Contract = contractsById[contractId]
    const answers = answersByContractId[contractId]
    return calculateUserMetrics(contract, bets, user, answers)
  })
}

export const calculateUserMetrics = (
  contract: Contract,
  bets: Bet[],
  user: DisplayUser,
  answers: Answer[]
) => {
  // ContractMetrics will have an answerId for every answer, and a null for the overall metrics.
  const currentMetrics = getContractBetMetricsPerAnswer(contract, bets, answers)

  return currentMetrics.map((current) => {
    return removeUndefinedProps({
      ...current,
      contractId: contract.id,
      userName: user.name,
      userId: user.id,
      userUsername: user.username,
      userAvatarUrl: user.avatarUrl,
      profitAdjustment: getAdjustedProfit(
        contract,
        current.profit,
        answers,
        current.answerId
      ),
    } as ContractMetric)
  })
}

```

## chart.ts

```ts
import { base64toPoints } from './og'
import { removeUndefinedProps } from './object'
import { first, last, mapValues, meanBy } from 'lodash'

export type Point<X, Y, T = unknown> = { x: X; y: Y; obj?: T }
export type HistoryPoint<T = unknown> = Point<number, number, T>
export type DistributionPoint<T = unknown> = Point<number, number, T>
export type ValueKind = 'Ṁ' | 'percent' | 'amount' | 'spice' | 'sweepies'

export type MultiPoints = { [answerId: string]: HistoryPoint<never>[] }

/** answer  -> base 64 encoded */
export type MultiBase64Points = { [answerId: string]: string }

export type MultiSerializedPoints = { [answerId: string]: [number, number][] }
/** [x, y] */
export type SerializedPoint = Readonly<[number, number]>

export const unserializePoints = (points: SerializedPoint[]) => {
  return points.map(([x, y]) => removeUndefinedProps({ x, y }))
}

export const unserializeBase64Multi = (data: MultiBase64Points) => {
  return mapValues(data, (text) => base64toPoints(text))
}

export const serializeMultiPoints = (data: {
  [answerId: string]: HistoryPoint[]
}) => {
  return mapValues(data, (points) =>
    points.map(({ x, y }) => [x, y] as [number, number])
  )
}

export const maxMinBin = <P extends HistoryPoint>(
  points: P[],
  bins: number
) => {
  if (points.length < 2 || bins <= 0) return points

  const min = points[0].x
  const max = points[points.length - 1].x
  const binWidth = Math.ceil((max - min) / bins)

  // for each bin, get the max, min, and median in that bin
  // TODO: time-weighted average instead of median?
  const result = []
  let lastInBin = points[0]
  for (let i = 0; i < bins; i++) {
    const binStart = min + i * binWidth
    const binEnd = binStart + binWidth
    const binPoints = points.filter((p) => p.x >= binStart && p.x < binEnd)
    if (binPoints.length === 0) {
      // insert a synthetic point at the start of the bin to prevent long diagonal lines
      result.push({ ...lastInBin, x: binEnd })
    } else if (binPoints.length <= 3) {
      lastInBin = binPoints[binPoints.length - 1]
      result.push(...binPoints)
    } else {
      lastInBin = binPoints[binPoints.length - 1]
      binPoints.sort((a, b) => a.y - b.y)
      const min = binPoints[0]
      const max = binPoints[binPoints.length - 1]
      const median = binPoints[Math.floor(binPoints.length / 2)]
      result.push(...[min, max, median].sort((a, b) => a.x - b.x))
    }
  }

  return result
}

export function binAvg<P extends HistoryPoint>(sorted: P[], limit = 100) {
  const length = sorted.length
  if (length <= limit) {
    return sorted
  }

  const min = first(sorted)?.x ?? 0
  const max = last(sorted)?.x ?? 0
  const binWidth = Math.ceil((max - min) / limit)

  const newPoints = []
  let lastAvgY = sorted[0].y

  for (let i = 0; i < limit; i++) {
    const binStart = min + i * binWidth
    const binEnd = binStart + binWidth
    const binPoints = sorted.filter((p) => p.x >= binStart && p.x < binEnd)
    if (binPoints.length > 0) {
      lastAvgY = meanBy(binPoints, 'y')
    }
    newPoints.push({ x: binEnd, y: lastAvgY })
  }

  return newPoints
}

```

## constants.ts

```ts
import { escapeRegExp } from 'lodash'
import { DEV_CONFIG } from './dev'
import { EnvConfig, PROD_CONFIG } from './prod'

export const ENV = (process.env.NEXT_PUBLIC_FIREBASE_ENV ?? 'PROD') as
  | 'PROD'
  | 'DEV'

export const CONFIGS: { [env: string]: EnvConfig } = {
  PROD: PROD_CONFIG,
  DEV: DEV_CONFIG,
}

export const TWOMBA_ENABLED = true
export const TWOMBA_CASHOUT_ENABLED = true
export const PRODUCT_MARKET_FIT_ENABLED = false
export const SPICE_PRODUCTION_ENABLED = false
export const SPICE_TO_MANA_CONVERSION_RATE = 1
export const CASH_TO_MANA_CONVERSION_RATE = 100
export const MIN_CASH_DONATION = 25
export const MIN_SPICE_DONATION = 25000
export const CHARITY_FEE = 0.05
export const CASH_TO_CHARITY_DOLLARS = 1
export const SPICE_TO_CHARITY_DOLLARS = (1 / 1000) * (1 - CHARITY_FEE) // prize points -> dollars
export const NY_FL_CASHOUT_LIMIT = 5000
export const DOLLAR_PURCHASE_LIMIT = 5000

export const SPICE_NAME = 'Prize Point'
export const SWEEPIES_NAME = 'sweepcash'
export const SPICE_MARKET_TOOLTIP = `Prize market! Earn ${SPICE_NAME}s on resolution`
export const SWEEPIES_MARKET_TOOLTIP = `Sweepstakes market! Win real cash prizes.`
export const CASH_SUFFIX = '--cash'

export const TRADE_TERM = 'trade'
export const TRADED_TERM = 'traded'
export const TRADING_TERM = 'trading'
export const TRADER_TERM = 'trader'

export const ENV_CONFIG = CONFIGS[ENV]

export function isAdminId(id: string) {
  return ENV_CONFIG.adminIds.includes(id)
}

export function isModId(id: string) {
  return MOD_IDS.includes(id)
}
export const DOMAIN = ENV_CONFIG.domain
export const LOVE_DOMAIN = ENV_CONFIG.loveDomain
export const LOVE_DOMAIN_ALTERNATE = ENV_CONFIG.loveDomainAlternate
export const FIREBASE_CONFIG = ENV_CONFIG.firebaseConfig
export const PROJECT_ID = ENV_CONFIG.firebaseConfig.projectId
export const IS_PRIVATE_MANIFOLD = ENV_CONFIG.visibility === 'PRIVATE'

export const AUTH_COOKIE_NAME = `FBUSER_${PROJECT_ID.toUpperCase().replace(
  /-/g,
  '_'
)}`

// Manifold's domain or any subdomains thereof
export const CORS_ORIGIN_MANIFOLD = new RegExp(
  '^https?://(?:[a-zA-Z0-9\\-]+\\.)*' + escapeRegExp(ENV_CONFIG.domain) + '$'
)
// Manifold love domain or any subdomains thereof
export const CORS_ORIGIN_MANIFOLD_LOVE = new RegExp(
  '^https?://(?:[a-zA-Z0-9\\-]+\\.)*' +
    escapeRegExp(ENV_CONFIG.loveDomain) +
    '$'
)
// Manifold love domain or any subdomains thereof
export const CORS_ORIGIN_MANIFOLD_LOVE_ALTERNATE = new RegExp(
  '^https?://(?:[a-zA-Z0-9\\-]+\\.)*' +
    escapeRegExp(ENV_CONFIG.loveDomainAlternate) +
    '$'
)

export const CORS_ORIGIN_CHARITY = new RegExp(
  '^https?://(?:[a-zA-Z0-9\\-]+\\.)*' + escapeRegExp('manifund.org') + '$'
)

// Vercel deployments, used for testing.
export const CORS_ORIGIN_VERCEL = new RegExp(
  '^https?://[a-zA-Z0-9\\-]+' + escapeRegExp('mantic.vercel.app') + '$'
)
// Any localhost server on any port
export const CORS_ORIGIN_LOCALHOST = /^http:\/\/localhost:\d+$/

// TODO: These should maybe be part of the env config?
export const BOT_USERNAMES = [
  'TenShinoe908',
  'subooferbot',
  'pos',
  'v',
  'acc',
  'jerk',
  'snap',
  'ArbitrageBot',
  'MarketManagerBot',
  'Botlab',
  'JuniorBot',
  'ManifoldDream',
  'ManifoldBugs',
  'ACXBot',
  'JamesBot',
  'RyanBot',
  'trainbot',
  'runebot',
  'LiquidityBonusBot',
  '538',
  'FairlyRandom',
  'Anatolii',
  'JeremyK',
  'Botmageddon',
  'SmartBot',
  'ShifraGazsi',
  'NiciusBot',
  'Bot',
  'Mason',
  'VersusBot',
  'GPT4',
  'EntropyBot',
  'veat',
  'ms_test',
  'arb',
  'Turbot',
  'MetaculusBot',
  'burkebot',
  'Botflux',
  '7',
  'hyperkaehler',
  'NcyBot',
  'ithaca',
  'GigaGaussian',
  'BottieMcBotface',
  'Seldon',
  'OnePercentBot',
  'arrbit',
  'ManaMaximizer',
  'rita',
  'uhh',
  'ArkPoint',
  'EliBot',
  'manifestussy',
  'mirrorbot',
  'JakeBot',
  'loopsbot',
  'breezybot',
  'echo',
  'Sayaka',
  'cc7',
  'Yuna',
  'ManifoldLove',
  'chooterb0t',
  'bonkbot',
  'NermitBundaloy',
  'FirstBot',
  'bawt',
  'FireTheCEO',
  'JointBot',
  'WrenTec',
  'TigerMcBot',
  'Euclidean',
  'manakin',
  'LUCAtheory',
  'TunglBot',
  'timetraveler',
  'bayesianbot',
  'CharlesLienBot',
  'JaguarMcBot',
  'AImogus',
  'brake',
  'brontobot',
  'OracleBot',
  'spacedroplet',
  'AriZernerBot',
  'PV_bot',
]

export const MOD_IDS = [
  'qnIAzz9RamaodeiJSiGZO6xRGC63', // Agh
  'srFlJRuVlGa7SEJDM4cY9B5k4Lj2', //bayesian
  'EJQOCF3MfLTFYbhiKncrNefQDBz1', // chrisjbillington
  'MV9fTVHetcfp3h6CVYzpypIsbyN2', // CodeandSolder
  'HTbxWFlzWGeHUTiwZvvF0qm8W433', // Conflux
  '9dAaZrNSx5OT0su6rpusDoG9WPN2', // dglid
  '5XMvQhA3YgcTzyoJRiNqGWyuB9k2', // dreev
  '946iB1LqFIR06G7d8q89um57PHh2', // egroj
  'hqdXgp0jK2YMMhPs067eFK4afEH3', // Eliza
  'kbHiTAGBahXdX9Z4sW29JpNrB0l2', // Ernie
  'W4yEF6idSMcNWEVUquowziSCZFI3', // EvanDaniel
  '2VhlvfTaRqZbFn2jqxk2Am9jgsE2', // Gabrielle
  'cA1JupYR5AR8btHUs2xvkui7jA93', // Gen
  'YGZdZUSFQyM8j2YzPaBqki8NBz23', // jack
  'cgrBqe2O3AU4Dnng7Nc9wuJHLKb2', // jskf
  '4juQfJkFnwX9nws3dFOpz4gc1mi2', // jacksonpolack
  'XeQf3ygmrGM1MxdsE3JSlmq8vL42', // Jacy
  'eSqS9cD5mzYcP2o7FrST8aC5IWn2', // PlasmaBallin (previously JosephNoonan)
  'JlVpsgzLsbOUT4pajswVMr0ZzmM2', // Joshua
  '7HhTMy4xECaVKvl5MmEAfVUkRCS2', // KevinBurke
  'jO7sUhIDTQbAJ3w86akzncTlpRG2', // MichaelWheatley
  'lkkqZxiWCpOgtJ9ztJcAKz4d9y33', // NathanpmYoung
  'fSrex43BDjeneNZ4ZLfxllSb8b42', // NcyRocks
  'BgCeVUcOzkexeJpSPRNomWQaQaD3', // SemioticRivalry
  'KHX2ThSFtLQlau58hrjtCX7OL2h2', // shankypanky (stefanie)
  'hUM4SO8a8qhfqT1gEZ7ElTCGSEz2', // Stralor
  'tO4DwIsujySUwtSnrr2hnU1WJtJ3', // WieDan
]

export const MVP = ['Eliza', 'Gabrielle', 'jacksonpolack']

export const VERIFIED_USERNAMES = [
  'EliezerYudkowsky',
  'ScottAlexander',
  'Aella',
  'ZviMowshowitz',
  'GavrielK',
  'CGPGrey',
  'LexFridman',
  'patio11',
  'RichardHanania',
  'Qualy',
  'Roko',
  'JonathanBlow',
  'DwarkeshPatel',
  'ByrneHobart',
  'RobertWiblin',
  'KelseyPiper',
  'SpencerGreenberg',
  'PaulChristiano',
  'BuckShlegeris',
  'Natalia',
  'zero',
  'OzzieGooen',
  'OliverHabryka',
  'Alicorn',
  'RazibKhan',
  'JamesMedlock',
  'Writer',
  'GeorgeHotz',
  'ShayneCoplan',
  'SanghyeonSeo',
  'KatjaGrace',
  'EmmettShear',
  'CateHall',
  'RobertSKMiles',
  'TarekMansour',
  'DylanMatthews',
  'RobinHanson',
  'KevinRoose18ac',
  'KnowNothing',
  'SantaPawsSSB',
  'AndersSandberg',
  'JosephWeisenthal',
  'LawrenceLessig',
  'NatFriedman',
  'patrissimo',
  'postjawline',
  'MatthewYglesias',
  'BillyMcRascal',
  'kyootbot',
  'MaximLott',
  'liron',
  'LarsDoucet',
  'PeterWildeford',
  'SethWalder',
  'SneakySly',
  'ConorSen',
  'transmissions11',
  'DanHendrycks',
]

export const BANNED_TRADING_USER_IDS = [
  'zgCIqq8AmRUYVu6AdQ9vVEJN8On1', //firstuserhere aka _deleted_
  'LIBAoi7tpqeNLYM1xxJ1QJBQqW32', //lastuserhere
  'p3ADzwIUS3fk0ka80XYEE3OM3S32', //PC
  '4JuXgDx47xPagH5mcLDqLzUSN5g2', // BTE
]

export const PARTNER_USER_IDS: string[] = [
  'sTUV8ejuM2byukNZp7qKP2OKXMx2', // NFL
  'rFJu0EIdR6RP8d1vHKSh62pbnbH2', // SimonGrayson
  'cb6PJqGOSVPEUhprDHCKWWMuJqu1', // DanMan314
  'HTbxWFlzWGeHUTiwZvvF0qm8W433', // Conflux
  'YGZdZUSFQyM8j2YzPaBqki8NBz23', // jack
  'hDq0cvn68jbAUVd6aWIU9aSv9ZA2', // strutheo
  'OEbsAczmbBc4Sl1bacYZNPJLLLc2', // SirCryptomind
  'JlVpsgzLsbOUT4pajswVMr0ZzmM2', // Joshua
  'xQqqZqlgcoSxTgPe03BiXmVE2JJ2', // Soli
  'Iiok8KHMCRfUiwtMq1tl5PeDbA73', // Lion
  'SqOJYkeySMQjqP3UAypw6DxPx4Z2', // Shump
  'hqdXgp0jK2YMMhPs067eFK4afEH3', // Eliza
  'BgCeVUcOzkexeJpSPRNomWQaQaD3', // SemioticRivalry
  'X1xu1kvOxuevx09xuR2urWfzf7i1', // KeenenWatts
  '4juQfJkFnwX9nws3dFOpz4gc1mi2', // jacksonpolack
  '8WEiWcxUd7QLeiveyI8iqbSIffU2', // goblinodds
  'Iua2KQvL6KYcfGLGNI6PVeGkseo1', // Ziddletwix
  'GRaWlYn2fNah0bvr6OW28l28nFn1', // cash
  'ZKkL3lFRFaYfiaT9ZOdiv2iUJBM2', // mint
  'hRbPwezgxnat6GpJQxoFxq1xgUZ2', // AmmonLam
  'iPQVGUbwOfT3MmWIZs3JaruVzhV2', // Mugiwaraplus
  'k9gKj9BgTLN5tkqYztHeNoSpwyl1', // OnePieceExplained
  'foOeshHZOET3yMvRTMPINpnb8Bj2', // PunishedFurry
  'EBGhoFSxRtVBu4617SLZUe1FeJt1', // FranklinBaldo
  'GPlNcdBrcfZ3PiAfhnI9mQfHZbm1', // RemNi
  '4xOTMCIOkGesdJft50wVFZFb5IB3', // Tripping
  'hUM4SO8a8qhfqT1gEZ7ElTCGSEz2', // Stralor aka Pat Scott
  'srFlJRuVlGa7SEJDM4cY9B5k4Lj2', // Bayesian
  'H6b5PWELWfRV6HhyHAlCGq7yJJu2', // AndrewG
  'EJQOCF3MfLTFYbhiKncrNefQDBz1', // chrisjbillington
  '7HhTMy4xECaVKvl5MmEAfVUkRCS2', // KevinBurke
  'oPxjIzlvC5fRbGCaVgkvAiyoXBB2', // mattyb
]

export const NEW_USER_HERLPER_IDS = [
  'cgrBqe2O3AU4Dnng7Nc9wuJHLKb2', // jskf
  '2VhlvfTaRqZbFn2jqxk2Am9jgsE2', // Gabrielle
  '4juQfJkFnwX9nws3dFOpz4gc1mi2', // jacksonpolack
  'BgCeVUcOzkexeJpSPRNomWQaQaD3', // SemioticRivalry
  'rQPOELuW5zaapaNPnBYQBMoonk92', // Tumbles
  'igi2zGXsfxYPgB0DJTXVJVmwCOr2', // Austin
  'tlmGNz9kjXc2EteizMORes4qvWl2', // Stephen
  '0k1suGSJKVUnHbCPEhHNpgZPkUP2', // Sinclair
  'AJwLWoo3xue32XIiAVrL5SyR1WB2', // Ian
  'uglwf3YKOZNGjjEXKc5HampOFRE2', // D4vid
  'GRwzCexe5PM6ThrSsodKZT9ziln2', // Inga
  'cA1JupYR5AR8btHUs2xvkui7jA93', // Genzy
  'hUM4SO8a8qhfqT1gEZ7ElTCGSEz2', // Stralor
  'sA7V30Ic73XZtniboy2eKr6ekkn1', // MartinRandall
  'JlVpsgzLsbOUT4pajswVMr0ZzmM2', // Joshua
  'srFlJRuVlGa7SEJDM4cY9B5k4Lj2', // Bayesian
  'oPxjIzlvC5fRbGCaVgkvAiyoXBB2', // mattyb
  'Gg7t9vPD4WPD1iPgj9RUFLYTxgH2', // nikki
  'OdBj5DW6PbYtnImvybpyZzfhb133', // @jim
]

export const OPTED_OUT_OF_LEAGUES = [
  'vuI5upWB8yU00rP7yxj95J2zd952', // ManifoldPolitics
  '8lZo8X5lewh4hnCoreI7iSc0GxK2', // ManifoldAI
  'IPTOzEqrpkWmEzh6hwvAyY9PqFb2', // Manifold
  'tRZZ6ihugZQLXPf6aPRneGpWLmz1', // ManifoldLove
  'BhNkw088bMNwIFF2Aq5Gg9NTPzz1', // acc
  'JlVpsgzLsbOUT4pajswVMr0ZzmM2', // Joshua
  'oPxjIzlvC5fRbGCaVgkvAiyoXBB2', // mattyb
  'NndHcEmeJhPQ6n7e7yqAPa3Oiih2', //josh
]

export const HIDE_FROM_LEADERBOARD_USER_IDS = [
  'BhNkw088bMNwIFF2Aq5Gg9NTPzz1', // acc
  'tRZZ6ihugZQLXPf6aPRneGpWLmz1', // ManifoldLove
]

export const HOUSE_BOT_USERNAME = 'acc'

export function supabaseUserConsolePath(userId: string) {
  const tableId = ENV === 'DEV' ? 19247 : 25916
  return `https://supabase.com/dashboard/project/${ENV_CONFIG.supabaseInstanceId}/editor/${tableId}/?filter=id%3Aeq%3A${userId}`
}

export function supabasePrivateUserConsolePath(userId: string) {
  const tableId = ENV === 'DEV' ? 2189688 : 153495548
  return `https://supabase.com/dashboard/project/${ENV_CONFIG.supabaseInstanceId}/editor/${tableId}/?filter=id%3Aeq%3A${userId}`
}

export function supabaseConsoleContractPath(contractId: string) {
  const tableId = ENV === 'DEV' ? 19254 : 25924
  return `https://supabase.com/dashboard/project/${ENV_CONFIG.supabaseInstanceId}/editor/${tableId}?filter=id%3Aeq%3A${contractId}`
}

export function supabaseConsoleTxnPath(txnId: string) {
  const tableId = ENV === 'DEV' ? 20014 : 25940
  return `https://supabase.com/dashboard/project/${ENV_CONFIG.supabaseInstanceId}/editor/${tableId}?filter=id%3Aeq%3A${txnId}`
}

export const GOOGLE_PLAY_APP_URL =
  'https://play.google.com/store/apps/details?id=com.markets.manifold'
export const APPLE_APP_URL =
  'https://apps.apple.com/us/app/manifold-markets/id6444136749'

export const TEN_YEARS_SECS = 60 * 60 * 24 * 365 * 10

export const DESTINY_GROUP_SLUG = 'destinygg'
export const PROD_MANIFOLD_LOVE_GROUP_SLUG = 'manifoldlove-relationships'

export const RATING_GROUP_SLUGS = ['nonpredictive', 'unsubsidized']

export const GROUP_SLUGS_TO_IGNORE_IN_MARKETS_EMAIL = [
  'manifold-6748e065087e',
  'manifold-features-25bad7c7792e',
  'bugs',
  'manifold-leagues',
  ...RATING_GROUP_SLUGS,
  DESTINY_GROUP_SLUG,
  PROD_MANIFOLD_LOVE_GROUP_SLUG,
]

// - Hide markets from signed-out landing page
// - Hide from onboarding topic selector
// - De-emphasize markets in the very first feed items generated for new users
export const HIDE_FROM_NEW_USER_SLUGS = [
  'fun',
  'selfresolving',
  'experimental',
  'trading-bots',
  'gambling',
  'free-money',
  'mana',
  'whale-watching',
  'spam',
  'test',
  'no-resolution',
  'eto',
  'friend-stocks',
  'ancient-markets',
  'jokes',
  'planecrash',
  'glowfic',
  'all-stonks',
  'the-market',
  'nonpredictive-profits',
  'personal-goals',
  'personal',
  'rationalussy',
  'nsfw',
  'manifold-6748e065087e',
  'bugs',
  'new-years-resolutions-2024',
  'metamarkets',
  'metaforecasting',
  'death-markets',
  ...GROUP_SLUGS_TO_IGNORE_IN_MARKETS_EMAIL,
]

export const GROUP_SLUGS_TO_NOT_INTRODUCE_IN_FEED = [
  'rationalussy',
  'nsfw',
  'planecrash',
  'glowfic',
  'no-resolution',
  'the-market',
  'spam',
  'test',
  'eto',
  'friend-stocks',
  'testing',
  'all-stonks',
  PROD_MANIFOLD_LOVE_GROUP_SLUG,
]

export const EXTERNAL_REDIRECTS = ['/umami']

export const DISCORD_INVITE_LINK = 'https://discord.com/invite/eHQBNBqXuh'
export const DISCORD_BOT_INVITE_LINK =
  'https://discord.com/api/oauth2/authorize?client_id=1074829857537663098&permissions=328565385280&scope=bot%20applications.commands'

export const YES_GRAPH_COLOR = '#11b981'

export const RESERVED_PATHS = [
  '_next',
  'about',
  'ad',
  'add-funds',
  'ads',
  'analytics',
  'api',
  'browse',
  'calibration',
  'card',
  'cards',
  'career',
  'careers',
  'charity',
  'common',
  'contact',
  'contacts',
  'cowp',
  'create',
  'date-docs',
  'dashboard',
  'discord',
  'discord-bot',
  'dream',
  'embed',
  'facebook',
  'find',
  'github',
  'google',
  'group',
  'groups',
  'help',
  'home',
  'jobs',
  'leaderboard',
  'leaderboards',
  'league',
  'leagues',
  'link',
  'linkAccount',
  'links',
  'live',
  'login',
  'lootbox',
  'mana-auction',
  'manifest',
  'markets',
  'messages',
  'mtg',
  'news',
  'notifications',
  'og-test',
  'payments',
  'portfolio',
  'privacy',
  'profile',
  'public',
  'questions',
  'referral',
  'referrals',
  'send',
  'server-sitemap',
  'sign-in',
  'sign-in-waiting',
  'sitemap',
  'slack',
  'stats',
  'styles',
  'swipe',
  'team',
  'terms',
  'tournament',
  'tournaments',
  'twitch',
  'twitter',
  'umami',
  'user',
  'users',
  'versus',
  'web',
  'welcome',
]

export const MANA_PURCHASE_RATE_CHANGE_DATE = new Date('2024-05-16T18:20:00Z')

```

## contract-metric.ts

```ts
export type ContractMetric = {
  id: number
  contractId: string
  from:
    | {
        // Monthly is not updated atm bc it's not used
        [period: string]: {
          profit: number
          profitPercent: number
          invested: number
          prevValue: number
          value: number
        }
      }
    | undefined
  hasNoShares: boolean
  hasShares: boolean
  hasYesShares: boolean
  invested: number
  loan: number
  maxSharesOutcome: string | null
  payout: number
  profit: number
  profitPercent: number
  totalShares: {
    [outcome: string]: number
  }
  userId: string
  userUsername: string
  userName: string
  userAvatarUrl: string
  lastBetTime: number
  answerId: string | null
  profitAdjustment?: number
}

export type ContractMetricsByOutcome = Record<string, ContractMetric[]>

```

## contract.ts

```ts
import { JSONContent } from '@tiptap/core'
import { getDisplayProbability } from './calculate'
import { Topic } from 'common/group'
import { ChartAnnotation } from 'common/supabase/chart-annotations'
import { sum } from 'lodash'
import { Answer } from './answer'
import { Bet } from './bet'
import { getLiquidity } from './calculate-cpmm'
import { ContractComment } from './comment'
import { ContractMetric, ContractMetricsByOutcome } from './contract-metric'
import { ENV_CONFIG, isAdminId, isModId } from './envs/constants'
import { Fees } from './fees'
import { PollOption } from './poll-option'
import { formatMoney, formatPercent } from './util/format'
import { MINUTE_MS } from './util/time'
import { MarketTierType } from './tier'

/************************************************

supabase status: columns exist for
  slug: text
  creatorId: text
  question: text
  visibility: text
  mechanism: text
  outcomeType: text
  createdTime: timestamp (from millis)
  closeTime?: timestamp (from millis)
  resolutionTime?: timestamp (from millis)
  resolution?: text
  resolutionProbability?: numeric
  popularityScore: numeric
  importanceScore: numeric

any changes to the type of these columns in firestore will require modifying
the supabase trigger, or replication of contracts may fail!

*************************************************/

type AnyContractType =
  | (CPMM & Binary)
  | (CPMM & PseudoNumeric)
  | QuadraticFunding
  | (CPMM & Stonk)
  | CPMMMulti
  | (NonBet & BountiedQuestion)
  | (NonBet & Poll)
  | CPMMMultiNumeric

export type Contract<T extends AnyContractType = AnyContractType> = {
  id: string
  slug: string // auto-generated; must be unique

  creatorId: string
  creatorName: string
  creatorUsername: string
  creatorAvatarUrl?: string
  creatorCreatedTime?: number

  question: string
  description: string | JSONContent // More info about what the contract is about
  visibility: Visibility

  createdTime: number // Milliseconds since epoch
  lastUpdatedTime: number // Updated on any change to the market (metadata, bet, comment)
  lastBetTime?: number
  lastCommentTime?: number
  closeTime?: number // When no more trading is allowed
  deleted?: boolean // If true, don't show market anywhere.

  isResolved: boolean
  resolutionTime?: number // When the contract creator resolved the market
  resolution?: string
  resolutionProbability?: number
  resolverId?: string
  isSpicePayout?: boolean

  closeEmailsSent?: number

  volume: number
  volume24Hours: number
  elasticity: number

  collectedFees: Fees
  uniqueBettorCount: number
  uniqueBettorCountDay: number

  unlistedById?: string
  featuredLabel?: string
  isTwitchContract?: boolean

  coverImageUrl?: string
  isRanked?: boolean

  gptCommentSummary?: string

  marketTier?: MarketTierType

  token: ContractToken
  siblingContractId?: string

  // Manifold.love
  loverUserId1?: string // The user id's of the pair of lovers referenced in the question.
  loverUserId2?: string // The user id's of the pair of lovers referenced in the question.
  matchCreatorId?: string // The user id of the person who proposed the match.
  isLove?: boolean

  /** @deprecated - no more auto-subsidization */
  isSubsidized?: boolean // NOTE: not backfilled, undefined = true
  /** @deprecated - try to use group-contracts table instead */
  groupSlugs?: string[]
  /** @deprecated - not deprecated, only updated in native column though*/
  popularityScore: number
  /** @deprecated - not deprecated, only updated in native column though*/
  importanceScore: number
  /** @deprecated - not deprecated, only updated in native column though*/
  dailyScore: number
  /** @deprecated - not deprecated, only updated in native column though*/
  freshnessScore: number
  /** @deprecated - not deprecated, only updated in native column though*/
  conversionScore: number
  /** @deprecated - not deprecated, only updated in native column though*/
  viewCount: number
  /** @deprecated - not up-to-date */
  likedByUserCount?: number
} & T

export type ContractToken = 'MANA' | 'CASH'
export type CPMMContract = Contract & CPMM
export type CPMMMultiContract = Contract & CPMMMulti
export type CPMMNumericContract = Contract & CPMMMultiNumeric
export type MarketContract =
  | CPMMContract
  | CPMMMultiContract
  | CPMMNumericContract

export type BinaryContract = Contract & Binary
export type PseudoNumericContract = Contract & PseudoNumeric
export type QuadraticFundingContract = Contract & QuadraticFunding
export type StonkContract = Contract & Stonk
export type BountiedQuestionContract = Contract & BountiedQuestion
export type PollContract = Contract & Poll

export type BinaryOrPseudoNumericContract =
  | BinaryContract
  | PseudoNumericContract
  | StonkContract

export type CPMM = {
  mechanism: 'cpmm-1'
  pool: { [outcome: string]: number }
  p: number // probability constant in y^p * n^(1-p) = k
  totalLiquidity: number // for historical reasons, this the total subsidy amount added in Ṁ
  subsidyPool: number // current value of subsidy pool in Ṁ
  prob: number
  probChanges: {
    day: number
    week: number
    month: number
  }
}

export type NonBet = {
  mechanism: 'none'
}

export const NON_BETTING_OUTCOMES: OutcomeType[] = ['BOUNTIED_QUESTION', 'POLL']
export const NO_CLOSE_TIME_TYPES: OutcomeType[] = NON_BETTING_OUTCOMES.concat([
  'STONK',
])

/**
 * Implemented as a set of cpmm-1 binary contracts, one for each answer.
 * The mechanism is stored among the contract's answers, which each
 * reference this contract id.
 */
export type CPMMMulti = {
  mechanism: 'cpmm-multi-1'
  outcomeType: 'MULTIPLE_CHOICE'
  shouldAnswersSumToOne: boolean
  addAnswersMode?: add_answers_mode

  totalLiquidity: number // for historical reasons, this the total subsidy amount added in Ṁ
  subsidyPool: number // current value of subsidy pool in Ṁ
  specialLiquidityPerAnswer?: number // Special liquidity mode, where initial ante is copied into each answer's pool, with a min probability, and only one answer can resolve YES. shouldAnswersSumToOne must be false.

  // Answers chosen on resolution, with the weights of each answer.
  // Weights sum to 100 if shouldAnswersSumToOne is true. Otherwise, range from 0 to 100 for each answerId.
  resolutions?: { [answerId: string]: number }

  // NOTE: This field is stored in the answers table and must be denormalized to the client.
  answers: Answer[]
  sort?: SortType
}

export type CPMMMultiNumeric = {
  mechanism: 'cpmm-multi-1'
  outcomeType: 'NUMBER'
  shouldAnswersSumToOne: true
  addAnswersMode: 'DISABLED'
  max: number
  min: number

  totalLiquidity: number // for historical reasons, this the total subsidy amount added in Ṁ
  subsidyPool: number // current value of subsidy pool in Ṁ

  // Answers chosen on resolution, with the weights of each answer.
  // Weights sum to 100 if shouldAnswersSumToOne is true. Otherwise, range from 0 to 100 for each answerId.
  resolutions?: { [answerId: string]: number }

  // NOTE: This field is stored in the answers table and must be denormalized to the client.
  answers: Answer[]
  sort?: SortType
}

export type add_answers_mode = 'DISABLED' | 'ONLY_CREATOR' | 'ANYONE'

export type QuadraticFunding = {
  outcomeType: 'QUADRATIC_FUNDING'
  mechanism: 'qf'
  answers: any[]
  // Mapping of how much each user has contributed to the matching pool
  // Note: Our codebase assumes every contract has a pool, which is why this isn't just a constant
  pool: { M$: number }

  // Used when the funding round pays out
  resolution?: 'MKT' | 'CANCEL'
  resolutions?: { [outcome: string]: number } // Used for MKT resolution.
}

export type Binary = {
  outcomeType: 'BINARY'
  initialProbability: number
  resolutionProbability?: number // Used for BINARY markets resolved to MKT
  resolution?: resolution
}

export type PseudoNumeric = {
  outcomeType: 'PSEUDO_NUMERIC'
  min: number
  max: number
  isLogScale: boolean
  resolutionValue?: number

  // same as binary market; map everything to probability
  initialProbability: number
  resolutionProbability?: number
}

export type MultipleNumeric = {
  outcomeType: 'NUMBER'
  answers: Answer[]
  min: number
  max: number
  resolution?: string | 'MKT' | 'CANCEL'
  resolutions?: { [outcome: string]: number } // Used for MKT resolution.
}

export type Stonk = {
  outcomeType: 'STONK'
  initialProbability: number
}

export type BountiedQuestion = {
  outcomeType: 'BOUNTIED_QUESTION'
  totalBounty: number
  bountyLeft: number
  /** @deprecated */
  bountyTxns?: string[]

  // Special mode where bounty pays out automatically in proportion to likes over 48 hours.
  isAutoBounty?: boolean
}

export type Poll = {
  outcomeType: 'POLL'
  options: PollOption[]
  resolutions?: string[]
}

export type MultiContract = CPMMMultiContract | CPMMNumericContract

type AnyOutcomeType =
  | Binary
  | QuadraticFunding
  | Stonk
  | BountiedQuestion
  | Poll
  | MultipleNumeric
  | CPMMMulti
  | PseudoNumeric

export type OutcomeType = AnyOutcomeType['outcomeType']
export type resolution = 'YES' | 'NO' | 'MKT' | 'CANCEL'
export const RESOLUTIONS = ['YES', 'NO', 'MKT', 'CANCEL'] as const
export const CREATEABLE_OUTCOME_TYPES = [
  'BINARY',
  'MULTIPLE_CHOICE',
  'PSEUDO_NUMERIC',
  'STONK',
  'BOUNTIED_QUESTION',
  'POLL',
  'NUMBER',
] as const

export const CREATEABLE_NON_PREDICTIVE_OUTCOME_TYPES = [
  'POLL',
  'BOUNTIED_QUESTION',
]

export type CreateableOutcomeType = (typeof CREATEABLE_OUTCOME_TYPES)[number]

export const renderResolution = (resolution: string, prob?: number) => {
  return (
    {
      YES: 'YES',
      NO: 'NO',
      CANCEL: 'N/A',
      MKT: formatPercent(prob ?? 0),
    }[resolution] || resolution
  )
}

export function contractPathWithoutContract(
  creatorUsername: string,
  slug: string
) {
  return `/${creatorUsername}/${slug}`
}

export function contractUrl(contract: Contract) {
  return `https://${ENV_CONFIG.domain}${contractPath(contract)}`
}

export function contractPool(contract: Contract) {
  return contract.mechanism === 'cpmm-1'
    ? formatMoney(contract.totalLiquidity)
    : contract.mechanism === 'cpmm-multi-1'
    ? formatMoney(
        sum(
          contract.answers.map((a) =>
            getLiquidity({ YES: a.poolYes, NO: a.poolNo })
          )
        )
      )
    : 'Empty pool'
}

export const isBinaryMulti = (contract: Contract) =>
  contract.mechanism === 'cpmm-multi-1' &&
  contract.outcomeType !== 'NUMBER' &&
  contract.answers.length === 2 &&
  contract.addAnswersMode === 'DISABLED' &&
  contract.shouldAnswersSumToOne
// contract.createdTime > 1708574059795 // In case we don't want to convert pre-commit contracts

export const getMainBinaryMCAnswer = (contract: Contract) =>
  isBinaryMulti(contract) && contract.mechanism === 'cpmm-multi-1'
    ? contract.answers[0]
    : undefined

export const getBinaryMCProb = (prob: number, outcome: 'YES' | 'NO' | string) =>
  outcome === 'YES' ? prob : 1 - prob

export function getBinaryProbPercent(contract: BinaryContract) {
  return formatPercent(getDisplayProbability(contract))
}

export function tradingAllowed(contract: Contract, answer?: Answer) {
  return (
    !contract.isResolved &&
    (!contract.closeTime || contract.closeTime > Date.now()) &&
    contract.mechanism !== 'none' &&
    (!answer || !answer.resolution)
  )
}

export const MAX_QUESTION_LENGTH = 120
export const MAX_DESCRIPTION_LENGTH = 16000

export const CPMM_MIN_POOL_QTY = 0.01
export const MULTI_NUMERIC_BUCKETS_MAX = 50
export const MULTI_NUMERIC_CREATION_ENABLED = true

export type Visibility = 'public' | 'unlisted'
export const VISIBILITIES = ['public' /*, 'unlisted'*/] as const

export const SORTS = [
  { label: 'High %', value: 'prob-desc' },
  { label: 'Low %', value: 'prob-asc' },
  { label: 'Oldest', value: 'old' },
  { label: 'Newest', value: 'new' },
  { label: 'Trending', value: 'liquidity' },
  { label: 'A-Z', value: 'alphabetical' },
] as const

export type SortType = (typeof SORTS)[number]['value']

export const MINUTES_ALLOWED_TO_UNRESOLVE = 10

export function contractPath(contract: {
  creatorUsername: string
  slug: string
}) {
  return `/${contract.creatorUsername}/${contract.slug}`
}

export type ContractParams = {
  contract: Contract
  lastBetTime?: number
  pointsString?: string
  multiPointsString?: { [answerId: string]: string }
  comments: ContractComment[]
  userPositionsByOutcome: ContractMetricsByOutcome
  totalPositions: number
  totalBets: number
  topContractMetrics: ContractMetric[]
  relatedContracts: Contract[]
  chartAnnotations: ChartAnnotation[]
  topics: Topic[]
  dashboards: { slug: string; title: string }[]
  pinnedComments: ContractComment[]
  betReplies: Bet[]
  cash?: {
    contract: Contract
    lastBetTime?: number
    pointsString: string
    multiPointsString: { [answerId: string]: string }
    userPositionsByOutcome: ContractMetricsByOutcome
    totalPositions: number
    totalBets: number
  }
}

export type MaybeAuthedContractParams =
  | {
      state: 'authed'
      params: ContractParams
    }
  | {
      state: 'deleted'
    }

export const MAX_CPMM_PROB = 0.99
export const MIN_CPMM_PROB = 0.01
export const MAX_STONK_PROB = 0.95
export const MIN_STONK_PROB = 0.2

export const canCancelContract = (userId: string, contract: Contract) => {
  const createdRecently = (Date.now() - contract.createdTime) / MINUTE_MS < 15
  return createdRecently || isModId(userId) || isAdminId(userId)
}

export const isMarketRanked = (contract: Contract) =>
  contract.isRanked != false &&
  contract.visibility === 'public' &&
  contract.deleted !== true

export const PROFIT_CUTOFF_TIME = 1715805887741
export const DPM_CUTOFF_TIMESTAMP = '2023-08-01 18:06:58.813000 +00:00'
export const getAdjustedProfit = (
  contract: Contract,
  profit: number,
  answers: Answer[] | undefined,
  answerId: string | null
) => {
  if (contract.mechanism === 'cpmm-multi-1') {
    // Null answerId stands for the summary of all answer metrics
    if (!answerId) {
      return isMarketRanked(contract) &&
        contract.resolutionTime &&
        contract.resolutionTime <= PROFIT_CUTOFF_TIME &&
        contract.createdTime > Date.parse(DPM_CUTOFF_TIMESTAMP)
        ? 9 * profit
        : isMarketRanked(contract)
        ? undefined
        : -1 * profit
    }
    const answer = answers?.find((a) => a.id === answerId)
    if (!answer) {
      console.log(
        `answer with id ${answerId} not found, but is required for cpmm-multi-1 contract: ${contract.id}`
      )
      return undefined
    }
    return isMarketRanked(contract) &&
      answer.resolutionTime &&
      answer.resolutionTime <= PROFIT_CUTOFF_TIME &&
      contract.createdTime > Date.parse(DPM_CUTOFF_TIMESTAMP)
      ? 9 * profit
      : isMarketRanked(contract)
      ? undefined
      : -1 * profit
  }

  return isMarketRanked(contract) &&
    contract.resolutionTime &&
    contract.resolutionTime <= PROFIT_CUTOFF_TIME
    ? 9 * profit
    : isMarketRanked(contract)
    ? undefined
    : -1 * profit
}
```

## economy.ts

```ts
import {
  CREATEABLE_NON_PREDICTIVE_OUTCOME_TYPES,
  OutcomeType,
} from 'common/contract'
import { MarketTierType, tiers } from './tier'
import { TWOMBA_ENABLED } from 'common/envs/constants'

export const FIXED_ANTE = 1000
const BASE_ANSWER_COST = FIXED_ANTE / 10
const ANTES = {
  BINARY: FIXED_ANTE,
  MULTIPLE_CHOICE: BASE_ANSWER_COST, // Amount per answer.
  FREE_RESPONSE: BASE_ANSWER_COST, // Amount per answer.
  PSEUDO_NUMERIC: FIXED_ANTE * 2.5,
  STONK: FIXED_ANTE,
  BOUNTIED_QUESTION: 0,
  POLL: FIXED_ANTE / 10,
  NUMBER: FIXED_ANTE * 10,
}

export const getTieredAnswerCost = (marketTier: MarketTierType | undefined) => {
  return marketTier
    ? BASE_ANSWER_COST * 10 ** (tiers.indexOf(marketTier) - 1)
    : BASE_ANSWER_COST
}

export const MINIMUM_BOUNTY = 10000
export const MULTIPLE_CHOICE_MINIMUM_COST = 1000

export const getAnte = (
  outcomeType: OutcomeType,
  numAnswers: number | undefined
) => {
  const ante = ANTES[outcomeType as keyof typeof ANTES] ?? FIXED_ANTE

  if (outcomeType === 'MULTIPLE_CHOICE') {
    return Math.max(ante * (numAnswers ?? 0), MULTIPLE_CHOICE_MINIMUM_COST)
  }

  return ante
}
export const getTieredCost = (
  baseCost: number,
  tier: MarketTierType | undefined,
  outcomeType: OutcomeType
) => {
  if (CREATEABLE_NON_PREDICTIVE_OUTCOME_TYPES.includes(outcomeType)) {
    return baseCost
  }

  const tieredCost = tier
    ? baseCost * 10 ** (tiers.indexOf(tier) - 1)
    : baseCost

  if (outcomeType == 'NUMBER' && tier != 'basic' && tier != 'play') {
    return tieredCost / 10
  }

  return tieredCost
}

/* Sweeps bonuses */
export const KYC_VERIFICATION_BONUS_CASH = 1
export const BETTING_STREAK_SWEEPS_BONUS_AMOUNT = 0.05
export const BETTING_STREAK_SWEEPS_BONUS_MAX = 0.25

/* Mana bonuses */
export const STARTING_BALANCE = 100
// for sus users, i.e. multiple sign ups for same person
export const SUS_STARTING_BALANCE = 10
export const PHONE_VERIFICATION_BONUS = 1000

export const REFERRAL_AMOUNT = 1000

// bonuses disabled
export const NEXT_DAY_BONUS = 100 // Paid on day following signup
export const MARKET_VISIT_BONUS = 100 // Paid on first distinct 5 market visits
export const MARKET_VISIT_BONUS_TOTAL = 500
export const UNIQUE_BETTOR_BONUS_AMOUNT = 5
export const SMALL_UNIQUE_BETTOR_BONUS_AMOUNT = 1
export const UNIQUE_ANSWER_BETTOR_BONUS_AMOUNT = 5
export const UNIQUE_BETTOR_LIQUIDITY = 20
export const SMALL_UNIQUE_BETTOR_LIQUIDITY = 5
export const MAX_TRADERS_FOR_BIG_BONUS = 50
export const MAX_TRADERS_FOR_BONUS = 10000

export const SUBSIDY_FEE = 0

export const BETTING_STREAK_BONUS_AMOUNT = 50
export const BETTING_STREAK_BONUS_MAX = 250

export const BETTING_STREAK_RESET_HOUR = 7

export const MANACHAN_TWEET_COST = 2500
export const PUSH_NOTIFICATION_BONUS = 1000
export const BURN_MANA_USER_ID = 'SlYWAUtOzGPIYyQfXfvmHPt8eu22'

export const PaymentAmounts = [
  {
    mana: 1_000,
    priceInDollars: 13.99,
    bonusInDollars: 0,
  },
  {
    mana: 2_500,
    priceInDollars: 29.99,
    bonusInDollars: 0,
  },
  {
    mana: 10_000,
    priceInDollars: 109.99,
    bonusInDollars: 0,
  },
  {
    mana: 100_000,
    priceInDollars: 1_000,
    bonusInDollars: 0,
  },
  {
    mana: 1_000,
    priceInDollars: 5,
    originalPriceInDollars: 13.99,
    bonusInDollars: 0,
    newUsersOnly: true,
  },
  {
    mana: 5_000,
    priceInDollars: 20,
    originalPriceInDollars: 55.99,
    bonusInDollars: 0,
    newUsersOnly: true,
  },
]

export const PaymentAmountsGIDX = [
  {
    mana: 1_000,
    priceInDollars: 15,
    bonusInDollars: 10,
  },
  {
    mana: 2_500,
    priceInDollars: 30,
    bonusInDollars: 25,
  },
  {
    mana: 10_000,
    priceInDollars: 110,
    bonusInDollars: 100,
  },
  {
    mana: 100_000,
    priceInDollars: 1_000,
    bonusInDollars: 1000,
  },
  {
    mana: 1_000,
    originalPriceInDollars: 15,
    priceInDollars: 7,
    bonusInDollars: 10,
    newUsersOnly: true,
  },
  {
    mana: 5_000,
    originalPriceInDollars: 55,
    priceInDollars: 20,
    bonusInDollars: 40,
    newUsersOnly: true,
  },
]
export type PaymentAmount = (typeof PaymentAmounts)[number]

export const MANA_WEB_PRICES = TWOMBA_ENABLED
  ? PaymentAmountsGIDX
  : PaymentAmounts

export type WebManaAmounts = (typeof PaymentAmounts)[number]['mana']
// TODO: these prices should be a function of whether the user is sweepstakes verified or not
export const IOS_PRICES = TWOMBA_ENABLED
  ? [
      {
        mana: 1_000,
        priceInDollars: 14.99,
        bonusInDollars: 10,
        sku: 'mana_1000',
      },
      {
        mana: 2_500,
        priceInDollars: 35.99,
        bonusInDollars: 25,
        sku: 'mana_2500',
      },
      {
        mana: 10_000,
        priceInDollars: 142.99,
        bonusInDollars: 100,
        sku: 'mana_10000',
      },
      // No 1M option on ios: the fees are too high
    ]
  : [
      {
        mana: 10_000,
        priceInDollars: 14.99,
        bonusInDollars: 0,
        sku: 'mana_1000',
      },
      {
        mana: 25_000,
        priceInDollars: 35.99,
        bonusInDollars: 0,
        sku: 'mana_2500',
      },
      {
        mana: 100_000,
        priceInDollars: 142.99,
        bonusInDollars: 0,
        sku: 'mana_10000',
      },
      // No 1M option on ios: the fees are too high
    ]

export const SWEEPIES_CASHOUT_FEE = 0.05
export const MIN_CASHOUT_AMOUNT = 25

```

## fees.ts

```ts
import { addObjects } from './object'
import { TWOMBA_ENABLED } from './constants'

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

## liquidity-provisioning.ts

```ts
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
}

```

## math.ts

```ts
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
}
```

## matrix.ts

```ts
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
}

```

## new-bet.ts

```ts

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
import { addObjects, removeUndefinedProps } from './object'
import {
  floatingEqual,
  floatingGreaterEqual,
  floatingLesserEqual,
} from './math'
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
}

```

## new-contract.ts

```ts
import { JSONContent } from '@tiptap/core'
import { Answer } from './answer'
import { getMultiCpmmLiquidity } from './calculate-cpmm'
import { computeBinaryCpmmElasticityFromAnte } from './calculate-metrics'
import {
  Binary,
  BountiedQuestion,
  CPMM,
  CPMMMulti,
  CPMMMultiNumeric,
  CREATEABLE_OUTCOME_TYPES,
  Contract,
  NonBet,
  Poll,
  PseudoNumeric,
  Stonk,
  Visibility,
  add_answers_mode,
} from './contract'
import { PollOption } from './poll-option'
import { User } from './user'
import { removeUndefinedProps } from './util/object'
import { randomString } from './util/random'
import { MarketTierType } from './tier'

export const NEW_MARKET_IMPORTANCE_SCORE = 0.25

export function getNewContract(props: {
  id: string
  slug: string
  creator: User
  question: string
  outcomeType: (typeof CREATEABLE_OUTCOME_TYPES)[number]
  description: JSONContent
  initialProb: number
  ante: number
  closeTime: number | undefined
  visibility: Visibility
  coverImageUrl?: string

  // twitch
  isTwitchContract: boolean | undefined

  // used for numeric markets
  min: number
  max: number
  isLogScale: boolean
  answers: string[]
  addAnswersMode?: add_answers_mode | undefined
  shouldAnswersSumToOne?: boolean | undefined

  isAutoBounty?: boolean | undefined
  marketTier?: MarketTierType
  token: 'MANA' | 'CASH'
}) {
  const {
    id,
    slug,
    creator,
    question,
    outcomeType,
    description,
    initialProb,
    ante,
    closeTime,
    visibility,
    isTwitchContract,
    min,
    max,
    isLogScale,
    answers,
    addAnswersMode,
    shouldAnswersSumToOne,
    coverImageUrl,
    isAutoBounty,
    marketTier,
    token,
  } = props
  const createdTime = Date.now()

  const propsByOutcomeType = {
    BINARY: () => getBinaryCpmmProps(initialProb, ante),
    PSEUDO_NUMERIC: () =>
      getPseudoNumericCpmmProps(initialProb, ante, min, max, isLogScale),
    MULTIPLE_CHOICE: () =>
      getMultipleChoiceProps(
        id,
        creator.id,
        answers,
        addAnswersMode ?? 'DISABLED',
        shouldAnswersSumToOne ?? true,
        ante
      ),
    STONK: () => getStonkCpmmProps(initialProb, ante),
    BOUNTIED_QUESTION: () => getBountiedQuestionProps(ante, isAutoBounty),
    POLL: () => getPollProps(answers),
    NUMBER: () => getNumberProps(id, creator.id, min, max, answers, ante),
  }[outcomeType]()

  const contract: Contract = removeUndefinedProps({
    id,
    slug,
    ...propsByOutcomeType,

    creatorId: creator.id,
    creatorName: creator.name,
    creatorUsername: creator.username,
    creatorAvatarUrl: creator.avatarUrl,
    creatorCreatedTime: creator.createdTime,
    coverImageUrl,

    question: question.trim(),
    description,
    visibility,
    isResolved: false,
    createdTime,
    closeTime,
    dailyScore: 0,
    popularityScore: 0,
    importanceScore: NEW_MARKET_IMPORTANCE_SCORE,
    freshnessScore: 0,
    conversionScore: DEFAULT_CONVERSION_SCORE,
    uniqueBettorCount: 0,
    uniqueBettorCountDay: 0,
    viewCount: 0,
    lastUpdatedTime: createdTime,

    volume: 0,
    volume24Hours: 0,
    elasticity:
      propsByOutcomeType.mechanism === 'cpmm-1'
        ? computeBinaryCpmmElasticityFromAnte(ante)
        : 4.99,

    collectedFees: {
      creatorFee: 0,
      liquidityFee: 0,
      platformFee: 0,
    },

    isTwitchContract,
    marketTier,
    token,
  })
  if (visibility === 'unlisted') {
    contract.unlistedById = creator.id
  }

  return contract as Contract
}

/*
import { PHANTOM_ANTE } from './antes'
import { calcDpmInitialPool } from './calculate-dpm'
const getBinaryDpmProps = (initialProb: number, ante: number) => {
  const { sharesYes, sharesNo, poolYes, poolNo, phantomYes, phantomNo } =
    calcDpmInitialPool(initialProb, ante, PHANTOM_ANTE)

  const system: DPM & Binary = {
    mechanism: 'dpm-2',
    outcomeType: 'BINARY',
    initialProbability: initialProb / 100,
    phantomShares: { YES: phantomYes, NO: phantomNo },
    pool: { YES: poolYes, NO: poolNo },
    totalShares: { YES: sharesYes, NO: sharesNo },
    totalBets: { YES: poolYes, NO: poolNo },
  }

  return system
}
*/

const getBinaryCpmmProps = (initialProb: number, ante: number) => {
  const pool = { YES: ante, NO: ante }
  const p = initialProb / 100

  const system: CPMM & Binary = {
    mechanism: 'cpmm-1',
    outcomeType: 'BINARY',
    totalLiquidity: ante,
    subsidyPool: 0,
    initialProbability: p,
    p,
    pool: pool,
    prob: p,
    probChanges: { day: 0, week: 0, month: 0 },
  }

  return system
}

const getPseudoNumericCpmmProps = (
  initialProb: number,
  ante: number,
  min: number,
  max: number,
  isLogScale: boolean
) => {
  const system: CPMM & PseudoNumeric = {
    ...getBinaryCpmmProps(initialProb, ante),
    outcomeType: 'PSEUDO_NUMERIC',
    min,
    max,
    isLogScale,
  }

  return system
}
const getStonkCpmmProps = (initialProb: number, ante: number) => {
  const system: CPMM & Stonk = {
    ...getBinaryCpmmProps(initialProb, ante),
    outcomeType: 'STONK',
  }
  return system
}

export const VERSUS_COLORS = ['#4e46dc', '#e9a23b']

const getMultipleChoiceProps = (
  contractId: string,
  userId: string,
  answers: string[],
  addAnswersMode: add_answers_mode,
  shouldAnswersSumToOne: boolean,
  ante: number
) => {
  const isBinaryMulti =
    addAnswersMode === 'DISABLED' &&
    answers.length === 2 &&
    shouldAnswersSumToOne

  const answersWithOther = answers.concat(
    !shouldAnswersSumToOne || addAnswersMode === 'DISABLED' ? [] : ['Other']
  )
  const answerObjects = createAnswers(
    contractId,
    userId,
    addAnswersMode,
    shouldAnswersSumToOne,
    ante,
    answersWithOther,
    isBinaryMulti ? VERSUS_COLORS : undefined
  )
  const system: CPMMMulti = {
    mechanism: 'cpmm-multi-1',
    outcomeType: 'MULTIPLE_CHOICE',
    addAnswersMode: addAnswersMode ?? 'DISABLED',
    shouldAnswersSumToOne: shouldAnswersSumToOne ?? true,
    answers: answerObjects,
    totalLiquidity: ante,
    subsidyPool: 0,
  }

  return system
}

const getNumberProps = (
  contractId: string,
  userId: string,
  min: number,
  max: number,
  answers: string[],
  ante: number
) => {
  const answerObjects = createAnswers(
    contractId,
    userId,
    'DISABLED',
    true,
    ante,
    answers
  )
  const system: CPMMMultiNumeric = {
    mechanism: 'cpmm-multi-1',
    outcomeType: 'NUMBER',
    addAnswersMode: 'DISABLED',
    shouldAnswersSumToOne: true,
    answers: answerObjects,
    totalLiquidity: ante,
    subsidyPool: 0,
    max,
    min,
  }

  return system
}

function createAnswers(
  contractId: string,
  userId: string,
  addAnswersMode: add_answers_mode,
  shouldAnswersSumToOne: boolean,
  ante: number,
  answers: string[],
  colors?: string[]
) {
  const ids = answers.map(() => randomString())

  let prob = 0.5
  let poolYes = ante / answers.length
  let poolNo = ante / answers.length

  if (shouldAnswersSumToOne && answers.length > 1) {
    const n = answers.length
    prob = 1 / n
    // Maximize use of ante given constraint that one answer resolves YES and
    // the rest resolve NO.
    // Means that:
    //   ante = poolYes + (n - 1) * poolNo
    // because this pays out ante mana to winners in this case.
    // Also, cpmm identity for probability:
    //   1 / n = poolNo / (poolYes + poolNo)
    poolNo = ante / (2 * n - 2)
    poolYes = ante / 2

    // Naive solution that doesn't maximize liquidity:
    // poolYes = ante * prob
    // poolNo = ante * (prob ** 2 / (1 - prob))
  }

  const now = Date.now()

  return answers.map((text, i) => {
    const id = ids[i]
    const answer: Answer = removeUndefinedProps({
      id,
      index: i,
      contractId,
      userId,
      text,
      createdTime: now,
      color: colors?.[i],

      poolYes,
      poolNo,
      prob,
      totalLiquidity: getMultiCpmmLiquidity({ YES: poolYes, NO: poolNo }),
      subsidyPool: 0,
      isOther:
        shouldAnswersSumToOne &&
        addAnswersMode !== 'DISABLED' &&
        i === answers.length - 1,
      probChanges: { day: 0, week: 0, month: 0 },
    })
    return answer
  })
}

const getBountiedQuestionProps = (
  ante: number,
  isAutoBounty: boolean | undefined
) => {
  const system: NonBet & BountiedQuestion = {
    mechanism: 'none',
    outcomeType: 'BOUNTIED_QUESTION',
    totalBounty: ante,
    bountyLeft: ante,
    isAutoBounty: isAutoBounty ?? false,
  }

  return system
}

const getPollProps = (answers: string[]) => {
  const ids = answers.map(() => randomString())

  const options: PollOption[] = answers.map((answer, i) => ({
    id: ids[i],
    index: i,
    text: answer,
    votes: 0,
  }))

  const system: NonBet & Poll = {
    mechanism: 'none',
    outcomeType: 'POLL',
    options: options,
  }
  return system
}

export const DEFAULT_CONVERSION_SCORE_NUMERATOR = 2
export const DEFAULT_CONVERSION_SCORE_DENOMINATOR = 15
const DEFAULT_CONVERSION_SCORE =
  DEFAULT_CONVERSION_SCORE_NUMERATOR / DEFAULT_CONVERSION_SCORE_DENOMINATOR

```

## object.ts

```ts
import { isEqual, mapValues, union } from 'lodash'

export const removeUndefinedProps = <T extends object>(obj: T): T => {
  const newObj: any = {}

  for (const key of Object.keys(obj)) {
    if ((obj as any)[key] !== undefined) newObj[key] = (obj as any)[key]
  }

  return newObj
}
export const removeNullOrUndefinedProps = <T extends object>(
  obj: T,
  exceptions?: string[]
): T => {
  const newObj: any = {}

  for (const key of Object.keys(obj)) {
    if (
      ((obj as any)[key] !== undefined && (obj as any)[key] !== null) ||
      (exceptions ?? []).includes(key)
    )
      newObj[key] = (obj as any)[key]
  }
  return newObj
}

export const addObjects = <T extends { [key: string]: number }>(
  obj1: T,
  obj2: T
) => {
  const keys = union(Object.keys(obj1), Object.keys(obj2))
  const newObj = {} as any

  for (const key of keys) {
    newObj[key] = (obj1[key] ?? 0) + (obj2[key] ?? 0)
  }

  return newObj as T
}

export const subtractObjects = <T extends { [key: string]: number }>(
  obj1: T,
  obj2: T
) => {
  const keys = union(Object.keys(obj1), Object.keys(obj2))
  const newObj = {} as any

  for (const key of keys) {
    newObj[key] = (obj1[key] ?? 0) - (obj2[key] ?? 0)
  }

  return newObj as T
}

export const hasChanges = <T extends object>(obj: T, partial: Partial<T>) => {
  const currValues = mapValues(partial, (_, key: keyof T) => obj[key])
  return !isEqual(currValues, partial)
}

export const hasSignificantDeepChanges = <T extends object>(
  obj: T,
  partial: Partial<T>,
  epsilonForNumbers: number
): boolean => {
  const compareValues = (currValue: any, partialValue: any): boolean => {
    if (typeof currValue === 'number' && typeof partialValue === 'number') {
      return Math.abs(currValue - partialValue) > epsilonForNumbers
    }
    if (typeof currValue === 'object' && typeof partialValue === 'object') {
      return hasSignificantDeepChanges(
        currValue,
        partialValue,
        epsilonForNumbers
      )
    }
    return !isEqual(currValue, partialValue)
  }

  for (const key in partial) {
    if (Object.prototype.hasOwnProperty.call(partial, key)) {
      if (compareValues(obj[key], partial[key])) {
        return true
      }
    }
  }

  return false
}
```

## og.ts

```ts
// see https://vercel.com/docs/concepts/functions/edge-functions/edge-functions-api for restrictions

export type Point = { x: number; y: number }

export function base64toPoints(base64urlString: string) {
  const b64 = base64urlString.replace(/-/g, '+').replace(/_/g, '/')
  const bin = atob(b64)
  const u = Uint8Array.from(bin, (c) => c.charCodeAt(0))
  const f = new Float32Array(u.buffer)

  const points = [] as { x: number; y: number }[]
  for (let i = 0; i < f.length; i += 2) {
    points.push({ x: f[i], y: f[i + 1] })
  }
  return points
}

```

## payouts-fixed.ts

```ts
import { sumBy } from 'lodash'
import { Bet } from './bet'
import { getCpmmLiquidityPoolWeights } from './calculate-cpmm'
import {
  CPMMContract,
  CPMMMultiContract,
  CPMMNumericContract,
} from './contract'
import { LiquidityProvision } from './liquidity-provision'
import { Answer } from './answer'

export const getFixedCancelPayouts = (
  contract: CPMMContract | CPMMMultiContract | CPMMNumericContract,
  bets: Bet[],
  liquidities: LiquidityProvision[]
) => {
  const liquidityPayouts = liquidities.map((lp) => ({
    userId: lp.userId,
    payout: lp.amount,
  }))

  const payouts = bets.map((bet) => ({
    userId: bet.userId,
    // We keep the platform fee.
    payout: bet.amount - bet.fees.platformFee,
  }))

  // Creator pays back all creator fees for N/A resolution.
  const creatorFees = sumBy(bets, (b) => b.fees.creatorFee)
  payouts.push({
    userId: contract.creatorId,
    payout: -creatorFees,
  })

  return { payouts, liquidityPayouts }
}

export const getStandardFixedPayouts = (
  outcome: string,
  contract:
    | CPMMContract
    | (CPMMMultiContract & { shouldAnswersSumToOne: false }),
  bets: Bet[],
  liquidities: LiquidityProvision[]
) => {
  const winningBets = bets.filter((bet) => bet.outcome === outcome)

  const payouts = winningBets.map(({ userId, shares }) => ({
    userId,
    payout: shares,
  }))

  const liquidityPayouts =
    contract.mechanism === 'cpmm-1'
      ? getLiquidityPoolPayouts(contract, outcome, liquidities)
      : []

  return { payouts, liquidityPayouts }
}

export const getMultiFixedPayouts = (
  answers: Answer[],
  resolutions: { [answerId: string]: number },
  bets: Bet[],
  liquidities: LiquidityProvision[]
) => {
  const payouts = bets
    .map(({ userId, shares, answerId, outcome }) => {
      const weight = answerId ? resolutions[answerId] ?? 0 : 0
      const outcomeWeight = outcome === 'YES' ? weight : 1 - weight
      const payout = shares * outcomeWeight
      return {
        userId,
        payout,
      }
    })
    .filter(({ payout }) => payout !== 0)

  const liquidityPayouts = getMultiLiquidityPoolPayouts(
    answers,
    resolutions,
    liquidities
  )
  return { payouts, liquidityPayouts }
}

export const getIndependentMultiYesNoPayouts = (
  answer: Answer,
  outcome: string,
  bets: Bet[],
  liquidities: LiquidityProvision[]
) => {
  const winningBets = bets.filter((bet) => bet.outcome === outcome)

  const payouts = winningBets.map(({ userId, shares }) => ({
    userId,
    payout: shares,
  }))

  const resolution = outcome === 'YES' ? 1 : 0
  const liquidityPayouts = getIndependentMultiLiquidityPoolPayouts(
    answer,
    resolution,
    liquidities
  )

  return { payouts, liquidityPayouts }
}

export const getLiquidityPoolPayouts = (
  contract: CPMMContract,
  outcome: string,
  liquidities: LiquidityProvision[]
) => {
  const { pool, subsidyPool } = contract
  const finalPool = pool[outcome] + (subsidyPool ?? 0)
  if (finalPool < 1e-3) return []

  const weights = getCpmmLiquidityPoolWeights(liquidities)

  return Object.entries(weights).map(([providerId, weight]) => ({
    userId: providerId,
    payout: weight * finalPool,
  }))
}

export const getIndependentMultiLiquidityPoolPayouts = (
  answer: Answer,
  resolution: number,
  liquidities: LiquidityProvision[]
) => {
  const payout = resolution * answer.poolYes + (1 - resolution) * answer.poolNo
  const weightsByUser = getCpmmLiquidityPoolWeights(liquidities)
  return Object.entries(weightsByUser)
    .map(([userId, weight]) => ({
      userId,
      payout: weight * payout,
    }))
    .filter(({ payout }) => payout >= 1e-3)
}

export const getMultiLiquidityPoolPayouts = (
  answers: Answer[],
  resolutions: { [answerId: string]: number },
  liquidities: LiquidityProvision[]
) => {
  const totalPayout = sumBy(answers, (answer) => {
    const weight = resolutions[answer.id] ?? 0
    const { poolYes, poolNo } = answer
    return weight * poolYes + (1 - weight) * poolNo
  })
  const weightsByUser = getCpmmLiquidityPoolWeights(liquidities)
  return Object.entries(weightsByUser)
    .map(([userId, weight]) => ({
      userId,
      payout: weight * totalPayout,
    }))
    .filter(({ payout }) => payout >= 1e-3)
}

export const getMktFixedPayouts = (
  contract:
    | CPMMContract
    | (CPMMMultiContract & { shouldAnswersSumToOne: false }),
  bets: Bet[],
  liquidities: LiquidityProvision[],
  resolutionProbability: number
) => {
  const outcomeProbs = {
    YES: resolutionProbability,
    NO: 1 - resolutionProbability,
  }

  const payouts = bets.map(({ userId, outcome, shares }) => {
    const p = outcomeProbs[outcome as 'YES' | 'NO'] ?? 0
    const payout = p * shares
    return { userId, payout }
  })

  const liquidityPayouts =
    contract.mechanism === 'cpmm-1'
      ? getLiquidityPoolProbPayouts(contract, outcomeProbs, liquidities)
      : []

  return { payouts, liquidityPayouts }
}

export const getIndependentMultiMktPayouts = (
  answer: Answer,
  bets: Bet[],
  liquidities: LiquidityProvision[],
  resolutionProbability: number
) => {
  const outcomeProbs = {
    YES: resolutionProbability,
    NO: 1 - resolutionProbability,
  }

  const payouts = bets.map(({ userId, outcome, shares }) => {
    const p = outcomeProbs[outcome as 'YES' | 'NO'] ?? 0
    const payout = p * shares
    return { userId, payout }
  })

  const liquidityPayouts = getIndependentMultiLiquidityPoolPayouts(
    answer,
    resolutionProbability,
    liquidities
  )

  return { payouts, liquidityPayouts }
}

export const getLiquidityPoolProbPayouts = (
  contract: CPMMContract,
  outcomeProbs: { [outcome: string]: number },
  liquidities: LiquidityProvision[]
) => {
  const { pool, subsidyPool } = contract

  const weightedPool = sumBy(
    Object.keys(pool),
    (o) => pool[o] * (outcomeProbs[o] ?? 0)
  )
  const finalPool = weightedPool + (subsidyPool ?? 0)
  if (finalPool < 1e-3) return []

  const weights = getCpmmLiquidityPoolWeights(liquidities)

  return Object.entries(weights).map(([providerId, weight]) => ({
    userId: providerId,
    payout: weight * finalPool,
  }))
}
```

## payouts.ts

```ts
import { sumBy, groupBy, mapValues } from 'lodash'

import { Bet } from './bet'
import { Contract, CPMMContract, CPMMMultiContract } from './contract'
import { LiquidityProvision } from './liquidity-provision'
import {
  getFixedCancelPayouts,
  getIndependentMultiMktPayouts,
  getMktFixedPayouts,
  getMultiFixedPayouts,
  getStandardFixedPayouts,
  getIndependentMultiYesNoPayouts,
} from './payouts-fixed'
import { getProbability } from './calculate'
import { Answer } from './answer'

export type Payout = {
  userId: string
  payout: number
}

export const getLoanPayouts = (bets: Bet[]): Payout[] => {
  const betsWithLoans = bets.filter((bet) => bet.loanAmount)
  const betsByUser = groupBy(betsWithLoans, (bet) => bet.userId)
  const loansByUser = mapValues(betsByUser, (bets) =>
    sumBy(bets, (bet) => -(bet.loanAmount ?? 0))
  )
  return Object.entries(loansByUser).map(([userId, payout]) => ({
    userId,
    payout,
  }))
}

export const groupPayoutsByUser = (payouts: Payout[]) => {
  const groups = groupBy(payouts, (payout) => payout.userId)
  return mapValues(groups, (group) => sumBy(group, (g) => g.payout))
}

export type PayoutInfo = {
  payouts: Payout[]
  liquidityPayouts: Payout[]
}

export const getPayouts = (
  outcome: string | undefined,
  contract: Contract,
  bets: Bet[],
  liquidities: LiquidityProvision[],
  resolutions?: {
    [outcome: string]: number
  },
  resolutionProbability?: number,
  answer?: Answer | null | undefined,
  allAnswers?: Answer[]
): PayoutInfo => {
  if (contract.mechanism === 'cpmm-1') {
    const prob = getProbability(contract)
    return getFixedPayouts(
      outcome,
      contract,
      bets,
      liquidities,
      resolutionProbability ?? prob
    )
  }
  if (
    contract.mechanism === 'cpmm-multi-1' &&
    !contract.shouldAnswersSumToOne &&
    answer
  ) {
    return getIndependentMultiFixedPayouts(
      answer,
      outcome,
      contract as any,
      bets,
      liquidities,
      resolutionProbability ?? answer.prob
    )
  }
  if (contract.mechanism === 'cpmm-multi-1') {
    if (outcome === 'CANCEL') {
      return getFixedCancelPayouts(contract, bets, liquidities)
    }
    if (!resolutions) {
      throw new Error('getPayouts: resolutions required for cpmm-multi-1')
    }
    if (!allAnswers) {
      throw new Error('getPayouts: answers required for cpmm-multi-1')
    }
    // Includes equivalent of 'MKT' and 'YES/NO' resolutions.
    return getMultiFixedPayouts(allAnswers, resolutions, bets, liquidities)
  }
  throw new Error('getPayouts not implemented')
}

export const getFixedPayouts = (
  outcome: string | undefined,
  contract:
    | CPMMContract
    | (CPMMMultiContract & { shouldAnswersSumToOne: false }),
  bets: Bet[],
  liquidities: LiquidityProvision[],
  resolutionProbability: number
) => {
  switch (outcome) {
    case 'YES':
    case 'NO':
      return getStandardFixedPayouts(outcome, contract, bets, liquidities)
    case 'MKT':
      return getMktFixedPayouts(
        contract,
        bets,
        liquidities,
        resolutionProbability
      )
    default:
    case 'CANCEL':
      return getFixedCancelPayouts(contract, bets, liquidities)
  }
}

export const getIndependentMultiFixedPayouts = (
  answer: Answer,
  outcome: string | undefined,
  contract: CPMMMultiContract & { shouldAnswersSumToOne: true },
  bets: Bet[],
  liquidities: LiquidityProvision[],
  resolutionProbability: number
) => {
  const filteredLiquidities = liquidities
    .filter((l) => l.answerId === answer.id)
    // Also include liquidity that is not assigned to an answer, and divide it by the number of answers.
    .concat(
      liquidities
        .filter((l) => !l.answerId)
        .map((l) => ({ ...l, amount: l.amount / contract.answers.length }))
    )
  switch (outcome) {
    case 'YES':
    case 'NO':
      return getIndependentMultiYesNoPayouts(
        answer,
        outcome,
        bets,
        filteredLiquidities
      )
    case 'MKT':
      return getIndependentMultiMktPayouts(
        answer,
        bets,
        filteredLiquidities,
        resolutionProbability
      )
    default:
    case 'CANCEL':
      return getFixedCancelPayouts(contract, bets, filteredLiquidities)
  }
}

```

## tier.ts

```ts
import { MarketContract } from './contract'
import { getAnte, getTieredCost } from './economy'

// Array of tiers in order
export const tiers = ['play', 'basic', 'plus', 'premium', 'crystal'] as const

export type BinaryDigit = '0' | '1'

export type TierParamsType =
  `${BinaryDigit}${BinaryDigit}${BinaryDigit}${BinaryDigit}${BinaryDigit}`

// Derive the MarketTierType from the array
export type MarketTierType = (typeof tiers)[number]

export function getTierFromLiquidity(
  contract: MarketContract,
  liquidity: number
): MarketTierType {
  const { outcomeType } = contract

  let numAnswers = undefined
  if ('answers' in contract) {
    numAnswers = contract.answers.length
  }

  const ante = getAnte(outcomeType, numAnswers)

  // Iterate through the tiers from highest to lowest
  for (let i = tiers.length - 1; i >= 0; i--) {
    const tier = tiers[i]
    const tierLiquidity = getTieredCost(ante, tier, outcomeType)
    // Return the first tier where the liquidity is greater or equal to the tier's requirement
    if (liquidity >= tierLiquidity) {
      return tier as MarketTierType
    }
  }
  // Default to the lowest tier if none of the conditions are met
  return 'play'
}

```

## time.ts

```ts
import dayjs from 'dayjs'
import relativeTime from 'dayjs/plugin/relativeTime'

dayjs.extend(relativeTime)

export function fromNow(time: number) {
  return dayjs(time).fromNow()
}

const FORMATTER = new Intl.DateTimeFormat('default', {
  dateStyle: 'medium',
  timeStyle: 'medium',
})

export const formatTime = FORMATTER.format

export function formatTimeShort(time: number) {
  return dayjs(time).format('MMM D, h:mma')
}

export function formatJustTime(time: number) {
  return dayjs(time).format('h:mma')
}

export const getCountdownString = (endDate: Date, includeSeconds = false) => {
  const remainingTimeMs = endDate.getTime() - Date.now()
  const isPast = remainingTimeMs < 0

  const seconds = Math.floor(Math.abs(remainingTimeMs) / 1000)
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)
  const days = Math.floor(hours / 24)

  const hoursStr = `${hours % 24}h`
  const minutesStr = `${minutes % 60}m`
  const daysStr = days > 0 ? `${days}d` : ''
  const secondsStr = includeSeconds ? ` ${seconds % 60}s` : ''
  return `${
    isPast ? '-' : ''
  }${daysStr} ${hoursStr} ${minutesStr} ${secondsStr}`
}

export const getCountdownStringHoursMinutes = (endDate: Date) => {
  const remainingTimeMs = endDate.getTime() - Date.now()
  const isPast = remainingTimeMs < 0

  const seconds = Math.floor(Math.abs(remainingTimeMs) / 1000)
  const minutes = Math.floor(seconds / 60)
  const hours = Math.floor(minutes / 60)

  const hoursStr = `${hours % 24}h`
  const minutesStr = `${minutes % 60}m`

  return `${isPast ? '-' : ''} ${hoursStr} ${minutesStr}`
}

```

## updated-contracts-metric-core.ts

```ts
import {
  createSupabaseDirectClient,
  SupabaseDirectClient,
} from 'shared/supabase/init'
import { log } from 'shared/utils'
import { DAY_MS, MONTH_MS, WEEK_MS } from 'common/util/time'
import { Contract, CPMM } from './contract'
import { computeElasticity } from './calculate-metrics'
import { hasChanges } from './object'
import { chunk, groupBy, mapValues } from 'lodash'
import { LimitBet } from 'common/bet'
import { bulkUpdateData } from './supabase/utils'
import { convertAnswer } from 'common/supabase/contracts'
import { bulkUpdateAnswers } from './supabase/answers'

export async function updateContractMetricsCore() {
  const pg = createSupabaseDirectClient()
  log('Loading contract data...')
  const allContracts = await pg.map(
    `
    select data from contracts
    where (resolution_time is null or resolution_time > now() - interval '1 month')
    `,
    [],
    (r) => r.data as Contract
  )
  log(`Loaded ${allContracts.length} contracts.`)
  const chunks = chunk(allContracts, 1000)
  let i = 0
  for (const contracts of chunks) {
    const contractIds = contracts.map((c) => c.id)
    const answers = await pg.map(
      `select *
       from answers
       where contract_id = any ($1)`,
      [contractIds],
      convertAnswer
    )
    log(`Loaded ${answers.length} answers.`)

    const now = Date.now()
    const dayAgo = now - DAY_MS
    const weekAgo = now - WEEK_MS
    const monthAgo = now - MONTH_MS

    log('Loading current contract probabilities...')
    const currentContractProbs = await getCurrentProbs(pg, contractIds)
    const currentAnswerProbs = Object.fromEntries(
      answers.map((a) => [
        a.id,
        {
          resTime: a?.resolutionTime ?? null,
          resProb:
            a?.resolution === 'YES' ? 1 : a?.resolution === 'NO' ? 0 : null,
          poolProb: a.prob ?? 0.5,
        },
      ])
    )

    log('Loading historic contract probabilities...')
    const [dayAgoProbs, weekAgoProbs, monthAgoProbs] = await Promise.all(
      [dayAgo, weekAgo, monthAgo].map((t) => getBetProbsAt(pg, t, contractIds))
    )

    log('Loading volume...')
    const volume = await getVolumeSince(pg, dayAgo, contractIds)

    log('Loading unfilled limits...')
    const limits = await getUnfilledLimitOrders(pg, contractIds)

    log('Computing metric updates...')

    const contractUpdates: ({ id: string } & Partial<Contract>)[] = []

    const answerUpdates: {
      id: string
      probChanges: {
        day: number
        week: number
        month: number
      }
    }[] = []

    for (const contract of contracts) {
      let cpmmFields: Partial<CPMM> = {}
      if (contract.mechanism === 'cpmm-1') {
        const { poolProb, resProb, resTime } = currentContractProbs[contract.id]
        const prob = resProb ?? poolProb
        const key = `${contract.id} _`
        const dayAgoProb = dayAgoProbs[key] ?? poolProb
        const weekAgoProb = weekAgoProbs[key] ?? poolProb
        const monthAgoProb = monthAgoProbs[key] ?? poolProb
        cpmmFields = {
          prob,
          probChanges: {
            day: resTime && resTime <= dayAgo ? 0 : prob - dayAgoProb,
            week: resTime && resTime <= weekAgo ? 0 : prob - weekAgoProb,
            month: resTime && resTime <= monthAgo ? 0 : prob - monthAgoProb,
          },
        }
      } else if (contract.mechanism === 'cpmm-multi-1') {
        const contractAnswers = answers.filter(
          (a) => a.contractId === contract.id
        )
        for (const answer of contractAnswers) {
          const { poolProb, resProb, resTime } =
            contract.shouldAnswersSumToOne && contract.resolutions
              ? {
                  poolProb: currentAnswerProbs[answer.id].poolProb,
                  resProb: (contract.resolutions[answer.id] ?? 0) / 100,
                  resTime: contract.resolutionTime,
                }
              : currentAnswerProbs[answer.id]
          const prob = resProb ?? poolProb
          const key = `${contract.id} ${answer.id}`
          const dayAgoProb = dayAgoProbs[key] ?? poolProb
          const weekAgoProb = weekAgoProbs[key] ?? poolProb
          const monthAgoProb = monthAgoProbs[key] ?? poolProb
          const answerCpmmFields = {
            probChanges: {
              day: resTime && resTime <= dayAgo ? 0 : prob - dayAgoProb,
              week: resTime && resTime <= weekAgo ? 0 : prob - weekAgoProb,
              month: resTime && resTime <= monthAgo ? 0 : prob - monthAgoProb,
            },
          }
          if (hasChanges(answer, answerCpmmFields)) {
            answerUpdates.push({
              id: answer.id,
              probChanges: answerCpmmFields.probChanges,
            })
          }
        }
      }
      const elasticity = computeElasticity(limits[contract.id] ?? [], contract)
      const update = {
        volume24Hours: volume[contract.id] ?? 0,
        elasticity,
        ...cpmmFields,
      }

      if (hasChanges(contract, update)) {
        contractUpdates.push({ id: contract.id, ...update })
      }
    }

    await bulkUpdateData(pg, 'contracts', contractUpdates)

    i += contracts.length
    log(`Finished ${i}/${allContracts.length} contracts.`)

    log('Writing answer updates...')
    await bulkUpdateAnswers(pg, answerUpdates)

    log('Done.')
  }
}

const getUnfilledLimitOrders = async (
  pg: SupabaseDirectClient,
  contractIds: string[]
) => {
  const unfilledBets = await pg.manyOrNone(
    `select contract_id, data
    from contract_bets
    where (data->'limitProb')::numeric > 0
    and not contract_bets.is_filled
    and not contract_bets.is_cancelled
    and contract_id = any($1)`,
    [contractIds]
  )
  return mapValues(
    groupBy(unfilledBets, (r) => r.contract_id as string),
    (rows) => rows.map((r) => r.data as LimitBet)
  )
}

const getVolumeSince = async (
  pg: SupabaseDirectClient,
  since: number,
  contractIds: string[]
) => {
  return Object.fromEntries(
    await pg.map(
      `select contract_id, sum(abs(amount)) as volume
      from contract_bets
      where created_time >= millis_to_ts($1)
      and not is_redemption
      and contract_id = any($2)
       group by contract_id`,
      [since, contractIds],
      (r) => [r.contract_id as string, parseFloat(r.volume as string)]
    )
  )
}

const getCurrentProbs = async (
  pg: SupabaseDirectClient,
  contractIds: string[]
) => {
  return Object.fromEntries(
    await pg.map(
      `select
         id, resolution_time as res_time,
         get_cpmm_pool_prob(data->'pool', (data->>'p')::numeric) as pool_prob,
         case when resolution = 'YES' then 1
              when resolution = 'NO' then 0
              when resolution = 'MKT' then resolution_probability
              else null end as res_prob
      from contracts
      where mechanism = 'cpmm-1'
      and id = any($1)
      `,
      [contractIds],
      (r) => [
        r.id as string,
        {
          resTime: r.res_time != null ? Date.parse(r.res_time as string) : null,
          resProb: r.res_prob != null ? parseFloat(r.res_prob as string) : null,
          poolProb: parseFloat(r.pool_prob),
        },
      ]
    )
  )
}

const getBetProbsAt = async (
  pg: SupabaseDirectClient,
  when: number,
  contractIds: string[]
) => {
  return Object.fromEntries(
    await pg.map(
      `with probs_before as (
        select distinct on (contract_id, answer_id)
          contract_id, answer_id, prob_after as prob
        from contract_bets
        where created_time < millis_to_ts($1)
        and contract_id = any($2) 
        order by contract_id, answer_id, created_time desc
      ), probs_after as (
        select distinct on (contract_id, answer_id)
          contract_id, answer_id, prob_before as prob
        from contract_bets
        where created_time >= millis_to_ts($1)
        and contract_id = any($2)
        order by contract_id, answer_id, created_time
      )
      select
        coalesce(pa.contract_id, pb.contract_id) as contract_id,
        coalesce(pa.answer_id, pb.answer_id) as answer_id,
        coalesce(pa.prob, pb.prob) as prob
      from probs_after as pa
      full outer join probs_before as pb
        on pa.contract_id = pb.contract_id and pa.answer_id = pb.answer_id
      `,
      [when, contractIds],
      (r) => [
        `${r.contract_id} ${r.answer_id ?? '_'}`,
        parseFloat(r.prob as string),
      ]
    )
  )
}
```