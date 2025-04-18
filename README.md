# lottery
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CollateralizedLoan {

    address public lender; // The lender (who gives the loan)
    address public borrower; // The borrower (who takes the loan)
    uint256 public collateralAmount; // Amount of collateral deposited by borrower
    uint256 public loanAmount; // Amount of loan taken by borrower
    uint256 public interestRate; // Interest rate (as percentage)
    uint256 public loanDueDate; // When the loan is due
    uint256 public loanStartTime; // Timestamp when the loan started

    // State to track loan repayment status
    bool public loanRepaid = false;

    // Events for logging
    event LoanIssued(address borrower, uint256 loanAmount, uint256 collateralAmount);
    event LoanRepaid(address borrower, uint256 repaidAmount);
    event CollateralWithdrawn(address borrower);

    // Modifier to ensure only the borrower can repay the loan
    modifier onlyBorrower() {
        require(msg.sender == borrower, "Only the borrower can execute this.");
        _;
    }

    // Modifier to ensure loan is not yet repaid
    modifier notRepaid() {
        require(!loanRepaid, "Loan already repaid.");
        _;
    }

    // Set the loan details
    function setLoanDetails(address _borrower, uint256 _loanAmount, uint256 _collateralAmount, uint256 _interestRate, uint256 _duration) public {
        lender = msg.sender; // The person who deploys the contract is the lender
        borrower = _borrower; // Address of the borrower
        collateralAmount = _collateralAmount; // Collateral deposit
        loanAmount = _loanAmount; // Loan amount
        interestRate = _interestRate; // Interest rate in percentage
        loanStartTime = block.timestamp; // Time when loan starts
        loanDueDate = loanStartTime + _duration; // Loan due date based on duration

        emit LoanIssued(_borrower, _loanAmount, _collateralAmount);
    }

    // Borrower repays the loan
    function repayLoan() public payable onlyBorrower notRepaid {
        uint256 totalRepayment = calculateTotalRepayment();

        require(msg.value >= totalRepayment, "Insufficient amount to repay the loan.");

        // Check if the borrower is paying enough
        require(msg.value >= totalRepayment, "Insufficient repayment amount.");

        // Transfer the repayment to the lender
        payable(lender).transfer(msg.value);

        loanRepaid = true;

        emit LoanRepaid(borrower, msg.value);
    }

    // Calculate the total amount to be repaid (loan + interest)
    function calculateTotalRepayment() public view returns (uint256) {
        uint256 interest = (loanAmount * interestRate) / 100;
        return loanAmount + interest;
    }

    // If loan is repaid, the borrower can withdraw the collateral
    function withdrawCollateral() public onlyBorrower {
        require(loanRepaid, "Loan must be repaid before collateral withdrawal.");

        // Transfer collateral back to the borrower
        payable(borrower).transfer(collateralAmount);

        emit CollateralWithdrawn(borrower);
    }

    // Check if the loan is overdue
    function isLoanOverdue() public view returns (bool) {
        return block.timestamp > loanDueDate && !loanRepaid;
    }

    // Allow the lender to withdraw the collateral if the loan is overdue and not repaid
    function claimCollateral() public {
        require(msg.sender == lender, "Only the lender can claim collateral.");
        require(isLoanOverdue(), "Loan is not overdue.");
        
        // Transfer collateral to the lender
        payable(lender).transfer(collateralAmount);
    }

    // Deposit collateral by borrower (before taking the loan)
    function depositCollateral() public payable onlyBorrower {
        require(msg.value == collateralAmount, "Incorrect collateral amount.");
    }
}
