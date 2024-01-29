import { Meteor } from 'meteor/meteor';
import { Mongo } from 'meteor/mongo';
import { Accounts } from 'meteor/accounts-base';

// Define MongoDB collections
const Users = new Mongo.Collection('users');
const Loans = new Mongo.Collection('loans');
const Payments = new Mongo.Collection('payments');

// Define user roles
const ROLES = ['admin', 'borrower', 'lender'];

// Define user schema
const UserSchema = new SimpleSchema({
  email: { type: String, regEx: SimpleSchema.RegEx.Email },
  role: { type: String, allowedValues: ROLES },
});

Users.attachSchema(UserSchema);

// Define loan schema
const LoanSchema = new SimpleSchema({
  userId: { type: String },
  amount: { type: Number },
  status: { type: String, allowedValues: ['pending', 'approved', 'rejected'] },
});

Loans.attachSchema(LoanSchema);

// Define payment schema
const PaymentSchema = new SimpleSchema({
  loanId: { type: String },
  amount: { type: Number },
  userId: { type: String },
});

Payments.attachSchema(PaymentSchema);

// Define methods for loan management
Meteor.methods({
  'loans.request'(amount) {
    if (!this.userId || Users.findOne(this.userId).role !== 'borrower') {
      throw new Meteor.Error('not-authorized', 'Only borrowers can request loans');
    }

    Loans.insert({
      userId: this.userId,
      amount,
      status: 'pending',
    });
  },

  'loans.approve'(loanId) {
    if (!this.userId || Users.findOne(this.userId).role !== 'lender') {
      throw new Meteor.Error('not-authorized', 'Only lenders can approve loans');
    }

    Loans.update(loanId, { $set: { status: 'approved' } });
  },

  'loans.reject'(loanId) {
    if (!this.userId || Users.findOne(this.userId).role !== 'lender') {
      throw new Meteor.Error('not-authorized', 'Only lenders can reject loans');
    }

    Loans.update(loanId, { $set: { status: 'rejected' } });
  },

  'payments.confirm'(loanId, amount) {
    if (!this.userId || Users.findOne(this.userId).role !== 'lender') {
      throw new Meteor.Error('not-authorized', 'Only lenders can confirm payments');
    }

    Payments.insert({
      loanId,
      amount,
      userId: this.userId,
    });
  },
});

// Publish data to clients
Meteor.publish('userData', function () {
  if (!this.userId) {
    return this.ready();
  }

  return Users.find(this.userId);
});

Meteor.publish('loans', function () {
  if (!this.userId) {
    return this.ready();
  }

  const user = Users.findOne(this.userId);

  if (user.role === 'admin') {
    return Loans.find();
  } else if (user.role === 'borrower') {
    return Loans.find({ userId: this.userId });
  } else {
    const lenderLoans = Loans.find({ status: 'pending' });
    const borrowerLoans = Loans.find({ userId: this.userId, status: 'approved' });
    return [lenderLoans, borrowerLoans];
  }
});

Meteor.publish('payments', function () {
  if (!this.userId) {
    return this.ready();
  }

  const user = Users.findOne(this.userId);

  if (user.role === 'admin') {
    return Payments.find();
  } else if (user.role === 'lender') {
    const lenderLoans = Loans.find({ userId: this.userId, status: 'approved' });
    return Payments.find({ userId: this.userId, loanId: { $in: lenderLoans.fetch().map(loan => loan._id) } });
  } else {
    return Payments.find({ userId: this.userId });
  }
});

// Server startup
Meteor.startup(() => {
  // Create default admin user
  if (Meteor.users.find().count() === 0) {
    const adminUserId = Accounts.createUser({
      email: 'admin@mergerware.com',
      password: 'mergerware',
    });

    Users.insert({
      _id: adminUserId,
      email: 'admin@mergerware.com',
      role: 'admin',
    });
  }
});
