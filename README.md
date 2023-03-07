import {baseEnvData} from "../../config/_base";
import {clickOnButton, clickOnEl, inputText, clearAndTypeText} from "../../helper/methods";
import {verifyTextOfEl, verifyUrl} from "../../helper/asserts";
import {buttons, messages, urlEndpoints} from "../../helper/names";
import {closeIcon, createInvoice, errorMessage} from "../../helper/selectors";
import {testInvoice} from "../../helper/testData";

const header = 'Create New Invoice';

const createItems = (()=>{
    clickOnButton(buttons.addItemHeader)
    inputText(createInvoice.headerField, testInvoice.headerField)
    cy.get(createInvoice.addItem).click({force: true})
    clearAndTypeText(createInvoice.quantity, testInvoice.quantity)
    inputText(createInvoice.quantity, testInvoice.quantity)
    inputText(createInvoice.rate, testInvoice.rate)
    inputText(createInvoice.discount, testInvoice.discount)
    inputText(createInvoice.tax, testInvoice.tax)
})
describe('Create invoice', function () {

    beforeEach(() => {
        cy.apiLogin(baseEnvData.businessOwner.email, baseEnvData.businessOwner.password)
        cy.visit(urlEndpoints.invoiceEndpoint)
        cy.contains(buttons.createInvoice).then((btn) => {
            cy.wrap(btn).click()
        })
        cy.contains(header).should('be.visible')
    });

    it('Verify close icon and button Cancel', () => {
        clickOnEl(closeIcon)
        clickOnButton(buttons.createInvoice)
        clickOnButton(buttons.cancel)
    });

    it('Create invoice with existing client', () => {
        clickOnEl(createInvoice.clientField)
        cy.get(createInvoice.listOfClients).find("li").last().click()
        createItems()

        cy.intercept('GET', '/invoice/*').as('Invoice')
        clickOnButton(buttons.save)

        cy.wait('@Invoice').its('response.body').then((body) => {
            const invoiceId = body.payload._id
            const invoiceCode = body.payload.code
            const clientName: string = body.payload.client.name
            const clientCompany: string = body.payload.client.company
            const clientEmail: string = body.payload.client.email
            verifyUrl(invoiceId)
            cy.get(createInvoice.invoiceTitle).should("contain", invoiceCode)
            cy.get(createInvoice.clientInfo).invoke('text').then((text) => {
                expect(text).contains(clientName, '**Invoice contains correct client name**')
                expect(text).contains(clientCompany, '**Invoice contains correct client company name**')
                expect(text).contains(clientEmail, '**Invoice contains correct client email**')
            });
        });
    });

    it('Create invoice with a new client', () => {
        let clientName: string
        cy.createClientApi().then((value) => {
            clientName = value.name
        }).then(() => {
            clickOnEl(createInvoice.clientField)
            cy.get(createInvoice.listOfClients).find("li").last().invoke('text').then((text) => {
                expect(text).contains(clientName, '**Invoice contains correct client name**')
            });
            cy.get(createInvoice.listOfClients).find("li").last().click()
        })
        createItems()

        cy.intercept('GET', '/invoice/*').as('Invoice')
        clickOnButton(buttons.save)

        cy.wait('@Invoice').its('response.body').then((body) => {
            const invoiceId = body.payload._id
            const invoiceCode = body.payload.code
            const clientCompany = body.payload.client.company
            const clientEmail = body.payload.client.email
            verifyUrl(invoiceId)
            cy.get(createInvoice.invoiceTitle).should("contain", invoiceCode)
            cy.get(createInvoice.clientInfo).invoke('text').then((text) => {
                verifyTextOfEl('tr', clientName)
                verifyTextOfEl('tr', clientCompany)
                verifyTextOfEl('tr', clientEmail)
            });
        });
    });

    it('Verify that we can not create an invoice without a client', () => {
        clickOnButton(buttons.save)
        verifyTextOfEl(errorMessage.client, messages.errorMessages.requireClient)
    });
});
