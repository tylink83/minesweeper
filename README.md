# minesweeper
Remake of the classic game.

//
//  ContentView.swift
//  minesweeper 2.0
//
//  Created by Link, Ty - Student on 12/10/24.
//

import SwiftUI

struct Cell {
    var isMine: Bool
    var isRevealed: Bool = false
    var surroundingMines: Int = 0
    var isFlagged: Bool = false
}

enum Difficulty: String, CaseIterable, Identifiable {
    case easy = "Easy"
    case medium = "Medium"
    case hard = "Hard"
    
    var id: String { self.rawValue }
    
    var settings: (rows: Int, columns: Int, mines: Int) {
        switch self {
        case .easy: return (8, 8, 10)
        case .medium: return (12, 12, 20)
        case .hard: return (16, 16, 40)
        }
    }
}

class MinesweeperViewModel: ObservableObject {
    @Published var board: [[Cell]]
    @Published var gameOver: Bool = false
    @Published var flaggingMode: Bool = false
    @Published var difficulty: Difficulty
    
    private var rows: Int
    private var columns: Int
    private var mines: Int
    
    init(difficulty: Difficulty) {
        self.difficulty = difficulty
        let settings = difficulty.settings
        self.rows = settings.rows
        self.columns = settings.columns
        self.mines = settings.mines
        
        board = Array(
            repeating: Array(
                repeating: Cell(isMine: false),
                count: columns
            ),
            count: rows
        )
        placeMines(mines: mines)
        calculateSurroundingMines()
    }
    
    private func placeMines(mines: Int) {
        var placedMines = 0
        while placedMines < mines {
            let row = Int.random(in: 0..<rows)
            let column = Int.random(in: 0..<columns)
            
            if !board[row][column].isMine {
                board[row][column].isMine = true
                placedMines += 1
            }
        }
    }
    
    private func calculateSurroundingMines() {
        for row in 0..<rows {
            for column in 0..<columns {
                if board[row][column].isMine { continue }
                board[row][column].surroundingMines = countMinesAround(row: row, column: column)
            }
        }
    }
    
    private func countMinesAround(row: Int, column: Int) -> Int {
        var count = 0
        for x in -1...1 {
            for y in -1...1 {
                let newRow = row + x
                let newColumn = column + y
                if newRow >= 0, newRow < rows, newColumn >= 0, newColumn < columns {
                    if board[newRow][newColumn].isMine {
                        count += 1
                    }
                }
            }
        }
        return count
    }
    
    func handleCellTap(row: Int, column: Int) {
        if flaggingMode {
            toggleFlag(row: row, column: column)
            return
        }
        guard !board[row][column].isFlagged && !board[row][column].isRevealed else { return }
        
        board[row][column].isRevealed = true
        
        if board[row][column].isMine {
            gameOver = true
            revealAllMines()
        } else if board[row][column].surroundingMines == 0 {
            revealSurroundingCells(row: row, column: column)
        }
    }
    
    func toggleFlag(row: Int, column: Int) {
        if !board[row][column].isRevealed {
            board[row][column].isFlagged.toggle()
        }
    }
    
    private func revealAllMines() {
        for row in 0..<rows {
            for column in 0..<columns {
                if board[row][column].isMine {
                    board[row][column].isRevealed = true
                }
            }
        }
    }
    
    private func revealSurroundingCells(row: Int, column: Int) {
        for x in -1...1 {
            for y in -1...1 {
                let newRow = row + x
                let newColumn = column + y
                if newRow >= 0, newRow < rows, newColumn >= 0, newColumn < columns {
                    handleCellTap(row: newRow, column: newColumn)
                }
            }
        }
    }
    
    func resetGame() {
        gameOver = false
        let settings = difficulty.settings
        rows = settings.rows
        columns = settings.columns
        mines = settings.mines
        
        board = Array(
            repeating: Array(
                repeating: Cell(isMine: false),
                count: columns
            ),
            count: rows
        )
        placeMines(mines: mines)
        calculateSurroundingMines()
    }
}

struct MinesweeperView: View {
    @StateObject private var viewModel: MinesweeperViewModel
    @State private var selectedDifficulty: Difficulty = .easy
    
    init() {
        _viewModel = StateObject(wrappedValue: MinesweeperViewModel(difficulty: .easy))
    }
    
    var body: some View {
        VStack {
            Picker("Difficulty", selection: $selectedDifficulty) {
                ForEach(Difficulty.allCases) { difficulty in
                    Text(difficulty.rawValue).tag(difficulty)
                }
            }
            .pickerStyle(SegmentedPickerStyle())
            .onChange(of: selectedDifficulty) { newDifficulty in
                viewModel.difficulty = newDifficulty
                viewModel.resetGame()
            }
            .padding()
            
            Text(viewModel.gameOver ? "💥 Game Over! 💥" : "Minesweeper")
                .font(.largeTitle)
                .padding()
            
            ForEach(0..<viewModel.board.count, id: \.self) { row in
                HStack {
                    ForEach(0..<viewModel.board[row].count, id: \.self) { column in
                        let cell = viewModel.board[row][column]
                        CellView(cell: cell)
                            .onTapGesture {
                                viewModel.handleCellTap(row: row, column: column)
                            }
                            .animation(.easeInOut, value: cell.isRevealed)
                    }
                }
            }
            
            HStack {
                Button(action: { viewModel.flaggingMode.toggle() }) {
                    Text(viewModel.flaggingMode ? "🚩 Flag Mode" : "Tap Mode")
                        .padding()
                        .background(viewModel.flaggingMode ? Color.red : Color.green)
                        .foregroundColor(.white)
                        .cornerRadius(10)
                }
                
                if viewModel.gameOver {
                    Button(action: { viewModel.resetGame() }) {
                        Text("Restart")
                            .padding()
                            .background(Color.blue)
                            .foregroundColor(.white)
                            .cornerRadius(10)
                    }
                }
            }
        }
        .padding()
    }
}

struct CellView: View {
    let cell: Cell
    
    var body: some View {
        ZStack {
            Rectangle()
                .stroke(Color.gray)
                .background(
                    cell.isRevealed
                    ? (cell.isMine ? Color.red : Color.white)
                    : Color.gray
                )
                .opacity(cell.isRevealed ? 1.0 : 0.7)
            
            if cell.isRevealed {
                if cell.isMine {
                    Text("💣")
                        .font(.title)
                        .foregroundColor(.white)
                } else if cell.surroundingMines > 0 {
                    Text("\(cell.surroundingMines)")
                        .foregroundColor(.black)
                }
            } else if cell.isFlagged {
                Text("🚩")
                    .font(.title)
                    .foregroundColor(.blue)
            }
        }
        .frame(width: 40, height: 40)
    }
}

struct ContentView: View {
    var body: some View {
        MinesweeperView()
    }
}

#Preview {
    ContentView()
}
