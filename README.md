//
// PrivAnalyticsSim.swift
// Simulates combined privacy techniques: DP + local masking + aggregate sanity checks
// Swift 5+
//

import Foundation

// Simple dataset: rows are numeric attributes
struct Row { let id: Int; let values: [Double] }

struct PrivacyBudget {
    let totalEps: Double
    private(set) var used: Double = 0
    mutating func charge(_ eps: Double) -> Bool {
        if used + eps > totalEps { return false }
        used += eps
        return true
    }
    func remaining() -> Double { totalEps - used }
}

func laplaceNoise(scale: Double) -> Double {
    let u = Double.random(in: -0.5..<0.5)
    return -scale * (u >= 0 ? 1.0 : -1.0) * log(1 - 2*abs(u))
}

class PrivAnalyticsEngine {
    var data: [Row]
    var budget: PrivacyBudget
    init(data: [Row], epsBudget: Double) {
        self.data = data; self.budget = PrivacyBudget(totalEps: epsBudget)
    }
    // query: dp-sum across a column
    func dpSum(column: Int, eps: Double) -> Double? {
        guard budget.charge(eps) else { return nil }
        let s = data.reduce(0.0) { $0 + ($1.values.indices.contains(column) ? $1.values[column] : 0.0) }
        let sensitivity = 1.0 // assume each record contributes at most 1 after clipping
        let noise = laplaceNoise(scale: sensitivity/eps)
        return s + noise
    }
    // local-masked aggregation: simulate per-row local randomization then server averages (RAPPOR-like toy)
    func localRandomizedMean(column: Int, p: Double = 0.6) -> Double {
        // each client with prob p reports true value, else random noise. Server averages reported values and debiases
        var reported: [Double] = []
        for r in data {
            let v = r.values[column]
            if Double.random(in: 0...1) < p {
                reported.append(v)
            } else {
                reported.append(Double.random(in: 0...1))
            }
        }
        let avgReported = reported.reduce(0.0, +) / Double(reported.count)
        // debias: expected reported = p*avgTrue + (1-p)*0.5 => avgTrue = (avgReported - (1-p)*0.5)/p
        return (avgReported - (1-p)*0.5) / p
    }
    // sanity check function to detect outliers in aggregates
    func sanityCheck(aggregate: Double, column: Int) -> Bool {
        // simple check: aggregate should be within [min* n, max * n] with small margin
        let n = Double(data.count)
        let colVals = data.map { $0.values[column] }
        guard let minV = colVals.min(), let maxV = colVals.max() else { return false }
        let low = minV * n - 10.0
        let high = maxV * n + 10.0
        return aggregate >= low && aggregate <= high
    }
}

// Demo
func demoPrivAnalyticsSim() {
    print("=== PrivAnalyticsSim Demo ===")
    // generate synthetic dataset
    let rows = (0..<100).map { i in Row(id: i, values: [Double.random(in: 0...10), Double.random(in: 0...1)]) }
    let engine = PrivAnalyticsEngine(data: rows, epsBudget: 1.0)
    if let dpS = engine.dpSum(column: 0, eps: 0.3) {
        print("DP sum (eps 0.3):", String(format: "%.3f", dpS), "sanity:", engine.sanityCheck(aggregate: dpS, column: 0))
    } else {
        print("Budget exceeded for dpSum")
    }
    let localMean = engine.localRandomizedMean(column: 1, p: 0.7)
    print("Local randomized mean (debias):", String(format: "%.4f", localMean))
    print("Remaining epsilon:", String(format: "%.3f", engine.budget.remaining()))
}

demoPrivAnalyticsSim()
