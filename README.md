# invoice
digital invoice
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Invoice Manager</title>
  
  <!-- Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  
  <!-- Vue 3 -->
  <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
  
  <!-- FontAwesome Icons -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
  
  <!-- HTML2PDF -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
  
  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          colors: {
            brand: {
              50: '#eef2ff',
              100: '#e0e7ff',
              500: '#6366f1',
              600: '#4f46e5',
              700: '#4338ca',
            }
          }
        }
      }
    }
  </script>

  <style>
    @media print {
      body * {
        visibility: hidden;
      }
      #printable-area, #printable-area * {
        visibility: visible;
      }
      #printable-area {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
      }
      .no-print {
        display: none !important;
      }
    }
    
    /* Touch target optimizations */
    button, input, select, textarea {
      touch-action: manipulation;
    }
    
    /* Hide scrollbar for clean UI */
    .no-scrollbar::-webkit-scrollbar {
      display: none;
    }
    .no-scrollbar {
      -ms-overflow-style: none;
      scrollbar-width: none;
    }
  </style>
</head>
<body class="bg-gray-50 dark:bg-gray-900 text-gray-800 dark:text-gray-100 antialiased font-sans min-h-screen pb-20 md:pb-0">

  <div id="app" class="max-w-md mx-auto md:max-w-4xl min-h-screen flex flex-col bg-white dark:bg-gray-900 shadow-xl relative">

    <!-- Top App Bar -->
    <header class="no-print sticky top-0 z-30 bg-indigo-600 text-white px-4 py-3 flex items-center justify-between shadow-md">
      <div class="flex items-center gap-3">
        <button v-if="currentView !== 'dashboard'" @click="currentView = 'dashboard'" class="text-white p-1 rounded-full hover:bg-indigo-700">
          <i class="fa-solid fa-arrow-left text-lg"></i>
        </button>
        <h1 class="text-xl font-bold tracking-tight">
          {{ viewTitles[currentView] || 'Invoice Pro' }}
        </h1>
      </div>
      <div class="flex items-center gap-2">
        <button @click="toggleDarkMode" class="p-2 rounded-full hover:bg-indigo-700 text-indigo-100">
          <i :class="isDarkMode ? 'fa-solid fa-sun' : 'fa-solid fa-moon'" class="text-lg"></i>
        </button>
        <button v-if="currentView === 'dashboard'" @click="createNewInvoice" class="bg-white text-indigo-600 font-semibold px-3 py-1.5 rounded-lg shadow text-sm hover:bg-indigo-50 active:scale-95 transition">
          + New
        </button>
      </div>
    </header>

    <!-- Main Content Body -->
    <main class="flex-1 overflow-y-auto p-4">

      <!-- 1. DASHBOARD VIEW -->
      <div v-if="currentView === 'dashboard'" class="space-y-5">
        
        <!-- Summary Cards -->
        <div class="grid grid-cols-2 gap-3">
          <div class="bg-indigo-50 dark:bg-gray-800 p-4 rounded-2xl border border-indigo-100 dark:border-gray-700">
            <div class="text-xs font-semibold text-indigo-600 dark:text-indigo-400 uppercase tracking-wider">Total Paid</div>
            <div class="text-xl font-extrabold text-gray-900 dark:text-white mt-1">{{ currencySymbol }}{{ totalPaid.toFixed(2) }}</div>
          </div>
          <div class="bg-amber-50 dark:bg-gray-800 p-4 rounded-2xl border border-amber-100 dark:border-gray-700">
            <div class="text-xs font-semibold text-amber-600 dark:text-amber-400 uppercase tracking-wider">Pending</div>
            <div class="text-xl font-extrabold text-gray-900 dark:text-white mt-1">{{ currencySymbol }}{{ totalPending.toFixed(2) }}</div>
          </div>
        </div>

        <!-- Filter & Search Bar -->
        <div class="flex flex-col gap-2">
          <div class="relative">
            <i class="fa-solid fa-magnifying-glass absolute left-3 top-3 text-gray-400"></i>
            <input v-model="searchQuery" type="text" placeholder="Search invoice or client..." class="w-full pl-9 pr-4 py-2 bg-gray-100 dark:bg-gray-800 border-none rounded-xl text-sm focus:ring-2 focus:ring-indigo-500">
          </div>
          
          <div class="flex gap-2 overflow-x-auto no-scrollbar py-1">
            <button v-for="status in ['all', 'paid', 'pending', 'overdue']" 
                    :key="status"
                    @click="filterStatus = status"
                    :class="filterStatus === status ? 'bg-indigo-600 text-white' : 'bg-gray-100 dark:bg-gray-800 text-gray-600 dark:text-gray-300'"
                    class="px-3 py-1 rounded-full text-xs font-medium capitalize whitespace-nowrap transition">
              {{ status }}
            </button>
          </div>
        </div>

        <!-- Invoice List -->
        <div class="space-y-3">
          <h2 class="text-sm font-semibold text-gray-500 dark:text-gray-400 uppercase tracking-wider">Recent Invoices</h2>
          
          <div v-if="filteredInvoices.length === 0" class="text-center py-10 text-gray-400">
            <i class="fa-solid fa-file-invoice text-4xl mb-2 text-gray-300 dark:text-gray-600"></i>
            <p class="text-sm">No invoices found.</p>
          </div>

          <div v-for="inv in filteredInvoices" 
               :key="inv.id" 
               @click="viewInvoice(inv)"
               class="bg-white dark:bg-gray-800 p-4 rounded-2xl shadow-sm border border-gray-100 dark:border-gray-700 flex justify-between items-center cursor-pointer hover:border-indigo-300 transition">
            <div class="space-y-1">
              <div class="flex items-center gap-2">
                <span class="font-bold text-gray-900 dark:text-white text-base">#{{ inv.number }}</span>
                <span :class="statusBadgeClass(inv.status)" class="text-[10px] font-bold px-2 py-0.5 rounded-full uppercase">
                  {{ inv.status }}
                </span>
              </div>
              <div class="text-xs text-gray-500 dark:text-gray-400">{{ inv.clientName || 'Unassigned Client' }}</div>
              <div class="text-[11px] text-gray-400">Due: {{ inv.dueDate }}</div>
            </div>

            <div class="text-right">
              <div class="text-base font-bold text-gray-900 dark:text-white">{{ currencySymbol }}{{ calculateTotal(inv).toFixed(2) }}</div>
              <i class="fa-solid fa-chevron-right text-gray-300 dark:text-gray-600 text-xs mt-2"></i>
            </div>
          </div>
        </div>
      </div>

      <!-- 2. INVOICE EDITOR / FORM -->
      <div v-if="currentView === 'editor'" class="space-y-5">
        <div class="bg-white dark:bg-gray-800 p-4 rounded-2xl border border-gray-100 dark:border-gray-700 space-y-4">
          <h2 class="text-lg font-bold text-gray-900 dark:text-white">Invoice Details</h2>
          
          <div class="grid grid-cols-2 gap-3">
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Invoice #</label>
              <input v-model="activeInvoice.number" type="text" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
            </div>
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Status</label>
              <select v-model="activeInvoice.status" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
                <option value="pending">Pending</option>
                <option value="paid">Paid</option>
                <option value="overdue">Overdue</option>
              </select>
            </div>
          </div>

          <div class="grid grid-cols-2 gap-3">
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Issue Date</label>
              <input v-model="activeInvoice.issueDate" type="date" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
            </div>
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Due Date</label>
              <input v-model="activeInvoice.dueDate" type="date" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
            </div>
          </div>

          <!-- Client Selection -->
          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Client</label>
            <select v-model="activeInvoice.clientId" @change="onClientSelect" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
              <option value="">-- Select Client --</option>
              <option v-for="client in clients" :key="client.id" :value="client.id">{{ client.name }}</option>
            </select>
          </div>
        </div>

        <!-- Line Items -->
        <div class="bg-white dark:bg-gray-800 p-4 rounded-2xl border border-gray-100 dark:border-gray-700 space-y-4">
          <div class="flex justify-between items-center">
            <h2 class="text-lg font-bold text-gray-900 dark:text-white">Items</h2>
            <button @click="addItem" class="text-xs bg-indigo-50 dark:bg-gray-700 text-indigo-600 dark:text-indigo-300 font-semibold px-3 py-1.5 rounded-lg">
              + Add Item
            </button>
          </div>

          <div v-for="(item, index) in activeInvoice.items" :key="index" class="p-3 bg-gray-50 dark:bg-gray-700/50 rounded-xl space-y-2 relative border border-gray-100 dark:border-gray-600">
            <button @click="removeItem(index)" class="absolute top-2 right-2 text-red-500 p-1">
              <i class="fa-solid fa-trash text-xs"></i>
            </button>

            <div>
              <input v-model="item.description" placeholder="Item description" class="w-full bg-white dark:bg-gray-800 p-2 rounded-lg border border-gray-200 dark:border-gray-600 text-sm">
            </div>

            <div class="grid grid-cols-3 gap-2">
              <div>
                <label class="text-[10px] text-gray-400">Qty</label>
                <input v-model.number="item.quantity" type="number" min="1" class="w-full bg-white dark:bg-gray-800 p-2 rounded-lg border border-gray-200 dark:border-gray-600 text-sm">
              </div>
              <div>
                <label class="text-[10px] text-gray-400">Price</label>
                <input v-model.number="item.price" type="number" step="0.01" class="w-full bg-white dark:bg-gray-800 p-2 rounded-lg border border-gray-200 dark:border-gray-600 text-sm">
              </div>
              <div>
                <label class="text-[10px] text-gray-400">Total</label>
                <div class="p-2 text-sm font-bold text-gray-700 dark:text-gray-200">
                  {{ currencySymbol }}{{ ((item.quantity || 0) * (item.price || 0)).toFixed(2) }}
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Taxes & Discounts -->
        <div class="bg-white dark:bg-gray-800 p-4 rounded-2xl border border-gray-100 dark:border-gray-700 space-y-3">
          <div class="grid grid-cols-2 gap-3">
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Tax (%)</label>
              <input v-model.number="activeInvoice.taxRate" type="number" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
            </div>
            <div>
              <label class="block text-xs font-semibold text-gray-500 mb-1">Discount (%)</label>
              <input v-model.number="activeInvoice.discountRate" type="number" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
            </div>
          </div>
        </div>

        <!-- Action Buttons -->
        <div class="flex gap-2">
          <button @click="saveInvoice" class="flex-1 bg-indigo-600 text-white font-bold py-3 rounded-xl shadow hover:bg-indigo-700 transition">
            Save Invoice
          </button>
          <button v-if="activeInvoice.id" @click="deleteInvoice(activeInvoice.id)" class="bg-red-50 dark:bg-red-900/30 text-red-600 dark:text-red-400 px-4 py-3 rounded-xl font-bold">
            <i class="fa-solid fa-trash"></i>
          </button>
        </div>

      </div>

      <!-- 3. INVOICE PREVIEW / PRINT VIEW -->
      <div v-if="currentView === 'preview'" class="space-y-4">
        
        <!-- Action Toolbar -->
        <div class="no-print flex gap-2">
          <button @click="currentView = 'editor'" class="flex-1 bg-gray-100 dark:bg-gray-800 text-gray-700 dark:text-gray-200 font-semibold py-2.5 rounded-xl text-sm">
            <i class="fa-solid fa-pen mr-1"></i> Edit
          </button>
          <button @click="exportPDF" class="flex-1 bg-indigo-600 text-white font-semibold py-2.5 rounded-xl text-sm shadow hover:bg-indigo-700">
            <i class="fa-solid fa-file-pdf mr-1"></i> PDF / Print
          </button>
        </div>

        <!-- Invoice Printable Document -->
        <div id="printable-area" class="bg-white text-gray-800 p-6 rounded-2xl shadow-sm border border-gray-100 text-xs space-y-6">
          
          <!-- Header -->
          <div class="flex justify-between items-start border-b pb-4">
            <div>
              <h2 class="text-xl font-bold text-indigo-600 uppercase tracking-wide">{{ business.name || 'Your Business Name' }}</h2>
              <p class="text-gray-500 mt-1 whitespace-pre-line">{{ business.address }}</p>
              <p class="text-gray-500">{{ business.email }} | {{ business.phone }}</p>
            </div>
            <div class="text-right">
              <h3 class="text-lg font-extrabold text-gray-800">INVOICE</h3>
              <p class="text-gray-500 font-semibold">#{{ activeInvoice.number }}</p>
              <span :class="statusBadgeClass(activeInvoice.status)" class="inline-block mt-2 text-[10px] font-bold px-2.5 py-0.5 rounded-full uppercase">
                {{ activeInvoice.status }}
              </span>
            </div>
          </div>

          <!-- Billed To & Dates -->
          <div class="grid grid-cols-2 gap-4">
            <div>
              <div class="font-bold text-gray-400 uppercase text-[10px] mb-1">Billed To</div>
              <div class="font-bold text-gray-800 text-sm">{{ activeInvoice.clientName || 'N/A' }}</div>
              <div class="text-gray-500 whitespace-pre-line mt-0.5">{{ activeInvoice.clientAddress }}</div>
              <div class="text-gray-500">{{ activeInvoice.clientEmail }}</div>
            </div>
            <div class="text-right space-y-1">
              <div>
                <span class="font-bold text-gray-400 uppercase text-[10px]">Issue Date: </span>
                <span class="font-semibold text-gray-700">{{ activeInvoice.issueDate }}</span>
              </div>
              <div>
                <span class="font-bold text-gray-400 uppercase text-[10px]">Due Date: </span>
                <span class="font-semibold text-gray-700">{{ activeInvoice.dueDate }}</span>
              </div>
            </div>
          </div>

          <!-- Items Table -->
          <table class="w-full text-left border-collapse">
            <thead>
              <tr class="border-b border-gray-200 text-gray-400 uppercase text-[9px] font-bold">
                <th class="py-2">Description</th>
                <th class="py-2 text-center">Qty</th>
                <th class="py-2 text-right">Price</th>
                <th class="py-2 text-right">Amount</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-gray-100">
              <tr v-for="(item, i) in activeInvoice.items" :key="i">
                <td class="py-2.5 font-medium text-gray-800">{{ item.description }}</td>
                <td class="py-2.5 text-center text-gray-600">{{ item.quantity }}</td>
                <td class="py-2.5 text-right text-gray-600">{{ currencySymbol }}{{ (item.price || 0).toFixed(2) }}</td>
                <td class="py-2.5 text-right font-bold text-gray-800">{{ currencySymbol }}{{ ((item.quantity||0)*(item.price||0)).toFixed(2) }}</td>
              </tr>
            </tbody>
          </table>

          <!-- Totals Breakdown -->
          <div class="border-t border-gray-200 pt-3 flex justify-end">
            <div class="w-1/2 space-y-1.5 text-right">
              <div class="flex justify-between text-gray-500">
                <span>Subtotal</span>
                <span class="font-semibold">{{ currencySymbol }}{{ calculateSubtotal(activeInvoice).toFixed(2) }}</span>
              </div>
              <div v-if="activeInvoice.discountRate > 0" class="flex justify-between text-emerald-600">
                <span>Discount ({{ activeInvoice.discountRate }}%)</span>
                <span>-{{ currencySymbol }}{{ calculateDiscount(activeInvoice).toFixed(2) }}</span>
              </div>
              <div v-if="activeInvoice.taxRate > 0" class="flex justify-between text-gray-500">
                <span>Tax ({{ activeInvoice.taxRate }}%)</span>
                <span>+{{ currencySymbol }}{{ calculateTax(activeInvoice).toFixed(2) }}</span>
              </div>
              <div class="flex justify-between text-sm font-bold text-gray-900 border-t border-gray-200 pt-2">
                <span>Total</span>
                <span class="text-indigo-600">{{ currencySymbol }}{{ calculateTotal(activeInvoice).toFixed(2) }}</span>
              </div>
            </div>
          </div>

          <!-- Notes / Terms -->
          <div class="border-t border-gray-100 pt-4 text-[10px] text-gray-400 space-y-1">
            <div class="font-bold text-gray-500 uppercase">Payment Terms & Notes</div>
            <p>{{ business.notes || 'Thank you for your business! Please remit payment within the due date.' }}</p>
          </div>

        </div>
      </div>

      <!-- 4. CLIENTS MANAGER -->
      <div v-if="currentView === 'clients'" class="space-y-4">
        <div class="flex justify-between items-center">
          <h2 class="text-base font-bold text-gray-900 dark:text-white">Client Directory</h2>
          <button @click="openClientModal()" class="bg-indigo-600 text-white text-xs font-semibold px-3 py-2 rounded-xl shadow">
            + New Client
          </button>
        </div>

        <div class="space-y-3">
          <div v-for="client in clients" :key="client.id" class="bg-white dark:bg-gray-800 p-4 rounded-2xl border border-gray-100 dark:border-gray-700 flex justify-between items-center">
            <div>
              <div class="font-bold text-gray-900 dark:text-white">{{ client.name }}</div>
              <div class="text-xs text-gray-500 dark:text-gray-400">{{ client.email }}</div>
              <div class="text-xs text-gray-400">{{ client.phone }}</div>
            </div>
            <button @click="openClientModal(client)" class="p-2 text-indigo-600">
              <i class="fa-solid fa-pen"></i>
            </button>
          </div>
        </div>
      </div>

      <!-- 5. SETTINGS VIEW -->
      <div v-if="currentView === 'settings'" class="space-y-4">
        <div class="bg-white dark:bg-gray-800 p-4 rounded-2xl border border-gray-100 dark:border-gray-700 space-y-3">
          <h2 class="text-base font-bold text-gray-900 dark:text-white">Business Info</h2>
          
          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Company Name</label>
            <input v-model="business.name" type="text" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          </div>

          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Email</label>
            <input v-model="business.email" type="email" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          </div>

          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Phone</label>
            <input v-model="business.phone" type="text" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          </div>

          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Address</label>
            <textarea v-model="business.address" rows="2" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm"></textarea>
          </div>

          <div>
            <label class="block text-xs font-semibold text-gray-500 mb-1">Default Currency Symbol</label>
            <input v-model="currencySymbol" type="text" class="w-20 bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm text-center font-bold">
          </div>

          <button @click="saveSettings" class="w-full bg-indigo-600 text-white font-bold py-2.5 rounded-xl shadow mt-2">
            Save Settings
          </button>
        </div>
      </div>

    </main>

    <!-- Bottom Navigation Bar (Android M3 style) -->
    <nav class="no-print fixed bottom-0 left-0 right-0 max-w-md mx-auto bg-white dark:bg-gray-800 border-t border-gray-200 dark:border-gray-700 flex justify-around py-2 z-30 shadow-lg">
      <button @click="currentView = 'dashboard'" :class="currentView === 'dashboard' ? 'text-indigo-600 dark:text-indigo-400' : 'text-gray-400'" class="flex flex-col items-center gap-1 w-1/4">
        <i class="fa-solid fa-chart-pie text-lg"></i>
        <span class="text-[10px] font-medium">Invoices</span>
      </button>

      <button @click="createNewInvoice" class="flex flex-col items-center gap-1 w-1/4 text-gray-400">
        <div class="bg-indigo-600 text-white p-2 rounded-full shadow-md -mt-5">
          <i class="fa-solid fa-plus text-base"></i>
        </div>
        <span class="text-[10px] font-medium text-indigo-600 dark:text-indigo-400">Create</span>
      </button>

      <button @click="currentView = 'clients'" :class="currentView === 'clients' ? 'text-indigo-600 dark:text-indigo-400' : 'text-gray-400'" class="flex flex-col items-center gap-1 w-1/4">
        <i class="fa-solid fa-users text-lg"></i>
        <span class="text-[10px] font-medium">Clients</span>
      </button>

      <button @click="currentView = 'settings'" :class="currentView === 'settings' ? 'text-indigo-600 dark:text-indigo-400' : 'text-gray-400'" class="flex flex-col items-center gap-1 w-1/4">
        <i class="fa-solid fa-gear text-lg"></i>
        <span class="text-[10px] font-medium">Settings</span>
      </button>
    </nav>

    <!-- Client Modal -->
    <div v-if="showClientModal" class="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4">
      <div class="bg-white dark:bg-gray-800 w-full max-w-sm rounded-2xl p-5 space-y-4 shadow-xl">
        <h3 class="text-base font-bold text-gray-900 dark:text-white">
          {{ modalClient.id ? 'Edit Client' : 'New Client' }}
        </h3>

        <div class="space-y-3">
          <input v-model="modalClient.name" placeholder="Client / Business Name" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          <input v-model="modalClient.email" placeholder="Email Address" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          <input v-model="modalClient.phone" placeholder="Phone Number" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm">
          <textarea v-model="modalClient.address" placeholder="Billing Address" rows="2" class="w-full bg-gray-50 dark:bg-gray-700 p-2.5 rounded-xl border border-gray-200 dark:border-gray-600 text-sm"></textarea>
        </div>

        <div class="flex gap-2 pt-2">
          <button @click="showClientModal = false" class="flex-1 bg-gray-100 dark:bg-gray-700 text-gray-600 dark:text-gray-300 font-semibold py-2 rounded-xl text-sm">
            Cancel
          </button>
          <button @click="saveClient" class="flex-1 bg-indigo-600 text-white font-semibold py-2 rounded-xl text-sm shadow">
            Save
          </button>
        </div>
      </div>
    </div>

  </div>

  <script>
    const { createApp, ref, computed, onMounted } = Vue;

    createApp({
      setup() {
        const currentView = ref('dashboard');
        const isDarkMode = ref(false);
        const searchQuery = ref('');
        const filterStatus = ref('all');
        const currencySymbol = ref('$');

        const viewTitles = {
          dashboard: 'Invoices',
          editor: 'Edit Invoice',
          preview: 'Invoice Preview',
          clients: 'Clients',
          settings: 'Settings'
        };

        // Business Info State
        const business = ref({
          name: 'Acme Digital Inc.',
          email: 'billing@acmedigital.com',
          phone: '+1 (555) 019-2834',
          address: '100 Innovation Way, Suite 400\nSan Francisco, CA 94105',
          notes: 'Payment is due within 15 days of invoice date. Late payments subject to a 1.5% monthly fee.'
        });

        // Clients Data State
        const clients = ref([
          { id: 'c1', name: 'Global Tech Ltd', email: 'ap@globaltech.com', phone: '+1 415 555 0122', address: '500 Market St, SF' },
          { id: 'c2', name: 'Apex Design Co', email: 'hello@apexdesign.io', phone: '+1 415 555 0199', address: '12 Mission St, SF' }
        ]);

        // Invoices Data State
        const invoices = ref([
          {
            id: 'inv_101',
            number: 'INV-1001',
            status: 'pending',
            issueDate: new Date().toISOString().split('T')[0],
            dueDate: new Date(Date.now() + 14*24*60*60*1000).toISOString().split('T')[0],
            clientId: 'c1',
            clientName: 'Global Tech Ltd',
            clientEmail: 'ap@globaltech.com',
            clientAddress: '500 Market St, SF',
            taxRate: 8.5,
            discountRate: 0,
            items: [
              { description: 'Mobile App UI/UX Design', quantity: 1, price: 1200.00 },
              { description: 'Frontend Development (Vue 3)', quantity: 10, price: 85.00 }
            ]
          }
        ]);

        const activeInvoice = ref({});
        const showClientModal = ref(false);
        const modalClient = ref({});

        // Calculation Helpers
        const calculateSubtotal = (inv) => {
          if (!inv || !inv.items) return 0;
          return inv.items.reduce((sum, item) => sum + ((item.quantity || 0) * (item.price || 0)), 0);
        };

        const calculateDiscount = (inv) => {
          const subtotal = calculateSubtotal(inv);
          return subtotal * ((inv.discountRate || 0) / 100);
        };

        const calculateTax = (inv) => {
          const subtotal = calculateSubtotal(inv);
          const discount = calculateDiscount(inv);
          return (subtotal - discount) * ((inv.taxRate || 0) / 100);
        };

        const calculateTotal = (inv) => {
          const subtotal = calculateSubtotal(inv);
          const discount = calculateDiscount(inv);
          const tax = calculateTax(inv);
          return subtotal - discount + tax;
        };

        // Filtered Invoices
        const filteredInvoices = computed(() => {
          return invoices.value.filter(inv => {
            const matchesSearch = inv.number.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
                                  (inv.clientName && inv.clientName.toLowerCase().includes(searchQuery.value.toLowerCase()));
            const matchesStatus = filterStatus.value === 'all' || inv.status === filterStatus.value;
            return matchesSearch && matchesStatus;
          });
        });

        const totalPaid = computed(() => {
          return invoices.value
            .filter(i => i.status === 'paid')
            .reduce((sum, i) => sum + calculateTotal(i), 0);
        });

        const totalPending = computed(() => {
          return invoices.value
            .filter(i => i.status === 'pending')
            .reduce((sum, i) => sum + calculateTotal(i), 0);
        });

        // Actions
        const createNewInvoice = () => {
          const nextNum = 1001 + invoices.value.length;
          activeInvoice.value = {
            id: 'inv_' + Date.now(),
            number: `INV-${nextNum}`,
            status: 'pending',
            issueDate: new Date().toISOString().split('T')[0],
            dueDate: new Date(Date.now() + 14*24*60*60*1000).toISOString().split('T')[0],
            clientId: '',
            clientName: '',
            clientEmail: '',
            clientAddress: '',
            taxRate: 0,
            discountRate: 0,
            items: [{ description: 'Consulting Service', quantity: 1, price: 100.00 }]
          };
          currentView.value = 'editor';
        };

        const viewInvoice = (inv) => {
          activeInvoice.value = JSON.parse(JSON.stringify(inv));
          currentView.value = 'preview';
        };

        const onClientSelect = () => {
          const client = clients.value.find(c => c.id === activeInvoice.value.clientId);
          if (client) {
            activeInvoice.value.clientName = client.name;
            activeInvoice.value.clientEmail = client.email;
            activeInvoice.value.clientAddress = client.address;
          }
        };

        const addItem = () => {
          activeInvoice.value.items.push({ description: '', quantity: 1, price: 0 });
        };

        const removeItem = (index) => {
          activeInvoice.value.items.splice(index, 1);
        };

        const saveInvoice = () => {
          const index = invoices.value.findIndex(i => i.id === activeInvoice.value.id);
          if (index > -1) {
            invoices.value[index] = { ...activeInvoice.value };
          } else {
            invoices.value.unshift({ ...activeInvoice.value });
          }
          saveToLocalStorage();
          currentView.value = 'preview';
        };

        const deleteInvoice = (id) => {
          invoices.value = invoices.value.filter(i => i.id !== id);
          saveToLocalStorage();
          currentView.value = 'dashboard';
        };

        const openClientModal = (client = null) => {
          if (client) {
            modalClient.value = { ...client };
          } else {
            modalClient.value = { id: '', name: '', email: '', phone: '', address: '' };
          }
          showClientModal.value = true;
        };

        const saveClient = () => {
          if (!modalClient.value.name) return;
          if (modalClient.value.id) {
            const idx = clients.value.findIndex(c => c.id === modalClient.value.id);
            if (idx > -1) clients.value[idx] = { ...modalClient.value };
          } else {
            modalClient.value.id = 'c_' + Date.now();
            clients.value.push({ ...modalClient.value });
          }
          saveToLocalStorage();
          showClientModal.value = false;
        };

        const exportPDF = () => {
          const element = document.getElementById('printable-area');
          if (typeof html2pdf !== 'undefined') {
            const opt = {
              margin:       0.5,
              filename:     `${activeInvoice.value.number || 'invoice'}.pdf`,
              image:        { type: 'jpeg', quality: 0.98 },
              html2canvas:  { scale: 2 },
              jsPDF:        { unit: 'in', format: 'letter', orientation: 'portrait' }
            };
            html2pdf().set(opt).from(element).save();
          } else {
            window.print();
          }
        };

        const toggleDarkMode = () => {
          isDarkMode.value = !isDarkMode.value;
          if (isDarkMode.value) {
            document.documentElement.classList.add('dark');
          } else {
            document.documentElement.classList.remove('dark');
          }
        };

        const statusBadgeClass = (status) => {
          switch (status) {
            case 'paid': return 'bg-emerald-100 text-emerald-700 dark:bg-emerald-900/40 dark:text-emerald-300';
            case 'pending': return 'bg-amber-100 text-amber-700 dark:bg-amber-900/40 dark:text-amber-300';
            case 'overdue': return 'bg-red-100 text-red-700 dark:bg-red-900/40 dark:text-red-300';
            default: return 'bg-gray-100 text-gray-700';
          }
        };

        const saveToLocalStorage = () => {
          localStorage.setItem('inv_app_invoices', JSON.stringify(invoices.value));
          localStorage.setItem('inv_app_clients', JSON.stringify(clients.value));
          localStorage.setItem('inv_app_business', JSON.stringify(business.value));
        };

        const loadLocalStorage = () => {
          const savedInvoices = localStorage.getItem('inv_app_invoices');
          const savedClients = localStorage.getItem('inv_app_clients');
          const savedBusiness = localStorage.getItem('inv_app_business');

          if (savedInvoices) invoices.value = JSON.parse(savedInvoices);
          if (savedClients) clients.value = JSON.parse(savedClients);
          if (savedBusiness) business.value = JSON.parse(savedBusiness);
        };

        const saveSettings = () => {
          saveToLocalStorage();
          alert('Settings saved successfully!');
        };

        onMounted(() => {
          loadLocalStorage();
        });

        return {
          currentView,
          viewTitles,
          isDarkMode,
          searchQuery,
          filterStatus,
          currencySymbol,
          business,
          clients,
          invoices,
          activeInvoice,
          filteredInvoices,
          totalPaid,
          totalPending,
          showClientModal,
          modalClient,
          createNewInvoice,
          viewInvoice,
          onClientSelect,
          addItem,
          removeItem,
          saveInvoice,
          deleteInvoice,
          openClientModal,
          saveClient,
          exportPDF,
          toggleDarkMode,
          statusBadgeClass,
          calculateSubtotal,
          calculateDiscount,
          calculateTax,
          calculateTotal,
          saveSettings
        };
      }
    }).mount('#app');
  </script>
</body>
</html>
